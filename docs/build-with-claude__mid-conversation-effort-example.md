---
title: 构建编排模式
url: https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-effort-example
description: 构建一个会话级模式，为多智能体扇出授予持续同意，并通过对话中途系统消息进行开启和关闭。
---

"Orchestration mode"（编排模式）是一个会话级开关：当它开启时，模型会对每个实质性请求投入最大程度的彻底性，先自行侦察任务，然后默认将工作扇出给并行的子智能体。当它关闭时，同一个编排工具会恢复为按请求逐次选择启用。

该模式不是一个 API 参数。它完全由已有文档记录的组件构建而成：

1. **一个努力级别：** 请求以文档中记录的 [Effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)（努力）值运行，例如 `xhigh`。在该页面列出的级别之上不存在隐藏级别。本示例在每个请求的顶层设置 effort，这不需要 beta 标头。
2. **一个模式提醒：** 一条 [mid-conversation system message](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)（对话中途系统消息）告知模型该模式已激活，每隔几轮发送一行简短的复习提醒，并在模式关闭时发送退出通知。顶层的 `system` 字段从不改变，因此缓存的前缀保持完整。
3. **工具描述中的持续同意：** 编排工具的描述声明，在模式开启期间，模型应为每个实质性任务编写并运行工作流，而无需事先询问。

<Note>
  本示例使用对话中途系统消息；有关支持它们的模型和平台，请参阅[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)。扇出本身会成倍增加令牌用量：单个请求可能会派生出许多子智能体对话，因此请将该模式保留给值得付出这一成本的工作。
</Note>

## 设置循环

该示例是单个文件。常量控制努力级别、扇出形态以及模式复习提醒的重新发送频率。`MAX_CONCURRENT` 限制同时运行的子智能体数量（PHP 移植版是顺序执行的，会忽略它）；`MAX_TOTAL_SUBTASKS` 限制模型在单次 Workflow 调用中可以排队的子任务数量。将两者分开可以让模型规划一个大型待办队列，而不必一次性全部启动。`DOC_TEST_MODE` 检查会在设置了该环境变量时将循环限制为单轮，以便自动化文档测试框架能够验证文件可以编译并快速完成，而无需运行完整的编排；您自己运行示例时请不要设置它。

<CodeGroup>
  ```python Python
  import atexit
  import concurrent.futures
  import hashlib
  import json
  import os
  import shutil
  import subprocess
  import sys
  import tempfile
  import threading

  import anthropic

  client = anthropic.Anthropic()

  MODEL = "claude-opus-5"
  EFFORT = "xhigh"

  SYSTEM_PROMPT = "You are a helpful general-purpose agent. Answer the user's request directly."

  REQUEST_TIMEOUT_SECONDS = 600
  BASH_TIMEOUT_SECONDS = 60
  TOOL_RESULT_MAX_CHARS = 8000
  MAX_CONCURRENT = 10
  DOC_TEST_MODE = bool(os.environ.get("DOC_TEST_MODE"))
  MAX_TOTAL_SUBTASKS = 2 if DOC_TEST_MODE else 200
  MAX_SUBAGENT_TURNS = 1 if DOC_TEST_MODE else 15
  MAX_MAIN_TURNS = 1 if DOC_TEST_MODE else 30
  TURNS_BETWEEN_REFRESHERS = 10
  JOURNAL_PATH = os.environ.get("ORCH_JOURNAL") or "orchestration_journal.json"
  ```

  ```typescript TypeScript
  import { exec } from "node:child_process";
  import { createHash } from "node:crypto";
  import { rmSync } from "node:fs";
  import { mkdtemp, readFile, rename, writeFile } from "node:fs/promises";
  import { tmpdir } from "node:os";
  import { join } from "node:path";
  import { promisify } from "node:util";

  import Anthropic from "@anthropic-ai/sdk";

  const client = new Anthropic();

  const MODEL = "claude-opus-5";
  const EFFORT = "xhigh";

  const SYSTEM_PROMPT =
    "You are a helpful general-purpose agent. Answer the user's request directly.";

  const REQUEST_TIMEOUT_SECONDS = 600;
  const BASH_TIMEOUT_SECONDS = 60;
  const TOOL_RESULT_MAX_CHARS = 8000;
  const MAX_CONCURRENT = 10;
  const DOC_TEST_MODE = Boolean(process.env.DOC_TEST_MODE);
  const MAX_TOTAL_SUBTASKS = DOC_TEST_MODE ? 2 : 200;
  const MAX_SUBAGENT_TURNS = DOC_TEST_MODE ? 1 : 15;
  const MAX_MAIN_TURNS = DOC_TEST_MODE ? 1 : 30;
  const TURNS_BETWEEN_REFRESHERS = 10;
  const JOURNAL_PATH = process.env.ORCH_JOURNAL || "orchestration_journal.json";
  ```

  ```csharp C#
  using System.Diagnostics;
  using System.Security.Cryptography;
  using System.Text;
  using System.Text.Json;
  using Anthropic;
  using Anthropic.Models.Messages;

  AnthropicClient client = new();

  const Model model = Model.ClaudeOpus5;
  var effort = Effort.Xhigh;

  const string systemPrompt = "You are a helpful general-purpose agent. Answer the user's request directly.";

  const int requestTimeoutSeconds = 600;
  // 其他移植版本使用 max_tokens 64000 进行流式传输。本版本使用非流式的
  // Messages.Create，而 API 会拒绝该大小的非流式请求。
  // 8192 是 Opus 4.0 和 4.1 的非流式上限，对于更新的 Opus 模型
  // 也是一个保守的选择。
  const int requestMaxTokens = 8192;
  const int bashTimeoutSeconds = 60;
  const int toolResultMaxChars = 8000;
  const int maxConcurrent = 10;
  var docTestMode = Environment.GetEnvironmentVariable("DOC_TEST_MODE") is { Length: > 0 };
  int maxTotalSubtasks = docTestMode ? 2 : 200;
  int maxSubagentTurns = docTestMode ? 1 : 15;
  int maxMainTurns = docTestMode ? 1 : 30;
  const int turnsBetweenRefreshers = 10;
  var journalPath = Environment.GetEnvironmentVariable("ORCH_JOURNAL") is { Length: > 0 } p ? p : "orchestration_journal.json";
  ```

  ```go Go
  import (
  	"bytes"
  	"cmp"
  	"context"
  	"crypto/sha256"
  	"encoding/hex"
  	"encoding/json"
  	"errors"
  	"fmt"
  	"log"
  	"os"
  	"os/exec"
  	"path/filepath"
  	"strings"
  	"sync"
  	"time"

  	"github.com/anthropics/anthropic-sdk-go"
  )

  var client = anthropic.NewClient()

  const (
  	modelID = anthropic.ModelClaudeOpus5
  	effort  = anthropic.OutputConfigEffortXhigh

  	systemPrompt = "You are a helpful general-purpose agent. Answer the user's request directly."

  	requestTimeoutSeconds  = 600
  	bashTimeoutSeconds     = 60
  	toolResultMaxChars     = 8000
  	maxConcurrent          = 10
  	turnsBetweenRefreshers = 10
  )

  var (
  	docTestMode      = os.Getenv("DOC_TEST_MODE") != ""
  	maxTotalSubtasks = ifTest(2, 200)
  	maxSubagentTurns = ifTest(1, 15)
  	maxMainTurns     = ifTest(1, 30)
  	journalPath      = cmp.Or(os.Getenv("ORCH_JOURNAL"), "orchestration_journal.json")
  )

  func ifTest(test, normal int) int {
  	if docTestMode {
  		return test
  	}
  	return normal
  }

  ```

  ```java Java
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.core.JsonValue;
  import com.anthropic.core.RequestOptions;
  import com.anthropic.helpers.MessageAccumulator;
  import com.anthropic.models.messages.ContentBlock;
  import com.anthropic.models.messages.ContentBlockParam;
  import com.anthropic.models.messages.Message;
  import com.anthropic.models.messages.MessageCreateParams;
  import com.anthropic.models.messages.MessageParam;
  import com.anthropic.models.messages.Model;
  import com.anthropic.models.messages.OutputConfig;
  import com.anthropic.models.messages.StopReason;
  import com.anthropic.models.messages.TextBlock;
  import com.anthropic.models.messages.Tool;
  import com.anthropic.models.messages.ToolBash20250124;
  import com.anthropic.models.messages.ToolResultBlockParam;
  import com.anthropic.models.messages.ToolUseBlock;
  import com.fasterxml.jackson.core.JsonProcessingException;
  import com.fasterxml.jackson.core.type.TypeReference;
  import com.fasterxml.jackson.databind.JsonNode;
  import com.fasterxml.jackson.databind.ObjectMapper;
  import java.io.IOException;
  import java.io.UncheckedIOException;
  import java.nio.charset.StandardCharsets;
  import java.nio.file.Files;
  import java.nio.file.Path;
  import java.nio.file.StandardCopyOption;
  import java.security.MessageDigest;
  import java.time.Duration;
  import java.util.ArrayList;
  import java.util.Comparator;
  import java.util.HashMap;
  import java.util.HexFormat;
  import java.util.List;
  import java.util.Map;
  import java.util.Objects;
  import java.util.Optional;
  import java.util.concurrent.Callable;
  import java.util.concurrent.CancellationException;
  import java.util.concurrent.CompletableFuture;
  import java.util.concurrent.ExecutionException;
  import java.util.concurrent.ExecutorService;
  import java.util.concurrent.Executors;
  import java.util.concurrent.Future;
  import java.util.concurrent.TimeUnit;
  import java.util.concurrent.locks.ReentrantLock;
  import java.util.stream.Collectors;
  import java.util.stream.IntStream;

  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  static final Model MODEL = Model.CLAUDE_OPUS_5;
  static final boolean DOC_TEST_MODE =
          !Objects.requireNonNullElse(System.getenv("DOC_TEST_MODE"), "").isEmpty();
  static final OutputConfig.Effort EFFORT = OutputConfig.Effort.XHIGH;

  static final String SYSTEM_PROMPT =
          "You are a helpful general-purpose agent. Answer the user's request directly.";

  static final int REQUEST_TIMEOUT_SECONDS = 600;
  static final RequestOptions REQUEST_OPTIONS =
          RequestOptions.builder().timeout(Duration.ofSeconds(REQUEST_TIMEOUT_SECONDS)).build();
  static final int BASH_TIMEOUT_SECONDS = 60;
  static final int TOOL_RESULT_MAX_CHARS = 8000;
  static final int MAX_CONCURRENT = 10;
  static final int MAX_TOTAL_SUBTASKS = DOC_TEST_MODE ? 2 : 200;
  static final int MAX_SUBAGENT_TURNS = DOC_TEST_MODE ? 1 : 15;
  static final int MAX_MAIN_TURNS = DOC_TEST_MODE ? 1 : 30;
  static final int TURNS_BETWEEN_REFRESHERS = 10;
  static final Path JOURNAL_PATH = Path.of(Optional.ofNullable(System.getenv("ORCH_JOURNAL"))
          .filter(s -> !s.isEmpty()).orElse("orchestration_journal.json"));
  ```

  ```php PHP
  use Anthropic\Client;
  use Anthropic\Messages\TextBlock;
  use Anthropic\Messages\ToolUseBlock;

  $client = new Client();

  const MODEL = 'claude-opus-5';
  define('DOC_TEST_MODE', (string) getenv('DOC_TEST_MODE') !== '');
  const EFFORT = 'xhigh';

  const SYSTEM_PROMPT = 'You are a helpful general-purpose agent. Answer the user\'s request directly.';

  const REQUEST_TIMEOUT_SECONDS = 600;
  const BASH_TIMEOUT_SECONDS = 60;
  const TOOL_RESULT_MAX_CHARS = 8000;
  const MAX_CONCURRENT = 10;
  define('MAX_TOTAL_SUBTASKS', DOC_TEST_MODE ? 2 : 200);
  define('MAX_SUBAGENT_TURNS', DOC_TEST_MODE ? 1 : 15);
  define('MAX_MAIN_TURNS', DOC_TEST_MODE ? 1 : 30);
  const TURNS_BETWEEN_REFRESHERS = 10;
  define('JOURNAL_PATH', getenv('ORCH_JOURNAL') ?: 'orchestration_journal.json');
  ```

  ```ruby Ruby
  require "anthropic"
  require "digest"
  require "fileutils"
  require "json"
  require "open3"
  require "tmpdir"

  CLIENT = Anthropic::Client.new

  MODEL = "claude-opus-5"
  EFFORT = :xhigh

  SYSTEM_PROMPT = "You are a helpful general-purpose agent. Answer the user's request directly."

  REQUEST_TIMEOUT_SECONDS = 600
  BASH_TIMEOUT_SECONDS = 60
  TOOL_RESULT_MAX_CHARS = 8000
  MAX_CONCURRENT = 10
  DOC_TEST_MODE = !ENV["DOC_TEST_MODE"].to_s.empty?
  MAX_TOTAL_SUBTASKS = DOC_TEST_MODE ? 2 : 200
  MAX_SUBAGENT_TURNS = DOC_TEST_MODE ? 1 : 15
  MAX_MAIN_TURNS = DOC_TEST_MODE ? 1 : 30
  TURNS_BETWEEN_REFRESHERS = 10
  JOURNAL_PATH = ENV["ORCH_JOURNAL"].to_s.empty? ? "orchestration_journal.json" : ENV["ORCH_JOURNAL"]
  ```
</CodeGroup>

## 定义模式提醒

这些提醒刻意保持简短。它们切换模式并指向工具描述，重量级的指令存放在那里。完整文本在模式开启时发送一次，复习提醒仅在若干用户轮次之后重新发送，退出通知在模式关闭时发送一次。

<CodeGroup>
  ```python Python
  MODE_ENTER = (
      "Orchestration mode is on: optimize for the most exhaustive, correct answer rather than "
      "the fastest one. Use the Workflow tool on every substantive task, sized to the problem's "
      "natural decomposition rather than the maximum the tool allows. See the Workflow tool's "
      "description for standing consent, granularity guidance, and quality patterns. Work solo "
      "only on conversational or trivial turns."
  )
  MODE_REFRESH = (
      "Orchestration mode is still on. Use the Workflow tool; see its standing consent section."
  )
  MODE_EXIT = (
      "Orchestration mode is off. The Workflow tool's standard opt-in rule applies again."
  )
  ```

  ```typescript TypeScript
  const MODE_ENTER =
    "Orchestration mode is on: optimize for the most exhaustive, correct answer rather than " +
    "the fastest one. Use the Workflow tool on every substantive task, sized to the problem's " +
    "natural decomposition rather than the maximum the tool allows. See the Workflow tool's " +
    "description for standing consent, granularity guidance, and quality patterns. Work solo " +
    "only on conversational or trivial turns.";
  const MODE_REFRESH =
    "Orchestration mode is still on. Use the Workflow tool; see its standing consent section.";
  const MODE_EXIT =
    "Orchestration mode is off. The Workflow tool's standard opt-in rule applies again.";
  ```

  ```csharp C#
  const string modeEnter =
      "Orchestration mode is on: optimize for the most exhaustive, correct answer rather than "
      + "the fastest one. Use the Workflow tool on every substantive task, sized to the problem's "
      + "natural decomposition rather than the maximum the tool allows. See the Workflow tool's "
      + "description for standing consent, granularity guidance, and quality patterns. Work solo "
      + "only on conversational or trivial turns.";
  const string modeRefresh =
      "Orchestration mode is still on. Use the Workflow tool; see its standing consent section.";
  const string modeExit =
      "Orchestration mode is off. The Workflow tool's standard opt-in rule applies again.";
  ```

  ```go Go
  const (
  	modeEnter = "Orchestration mode is on: optimize for the most exhaustive, correct answer rather than " +
  		"the fastest one. Use the Workflow tool on every substantive task, sized to the problem's " +
  		"natural decomposition rather than the maximum the tool allows. See the Workflow tool's " +
  		"description for standing consent, granularity guidance, and quality patterns. Work solo " +
  		"only on conversational or trivial turns."
  	modeRefresh = "Orchestration mode is still on. Use the Workflow tool; see its standing consent section."
  	modeExit    = "Orchestration mode is off. The Workflow tool's standard opt-in rule applies again."
  )

  ```

  ```java Java
  static final String MODE_ENTER =
          "Orchestration mode is on: optimize for the most exhaustive, correct answer rather than "
                  + "the fastest one. Use the Workflow tool on every substantive task, sized to the problem's "
                  + "natural decomposition rather than the maximum the tool allows. See the Workflow tool's "
                  + "description for standing consent, granularity guidance, and quality patterns. Work solo "
                  + "only on conversational or trivial turns.";
  static final String MODE_REFRESH =
          "Orchestration mode is still on. Use the Workflow tool; see its standing consent section.";
  static final String MODE_EXIT =
          "Orchestration mode is off. The Workflow tool's standard opt-in rule applies again.";
  ```

  ```php PHP
  const MODE_ENTER =
      'Orchestration mode is on: optimize for the most exhaustive, correct answer rather than '
      . 'the fastest one. Use the Workflow tool on every substantive task, sized to the problem\'s '
      . 'natural decomposition rather than the maximum the tool allows. See the Workflow tool\'s '
      . 'description for standing consent, granularity guidance, and quality patterns. Work solo '
      . 'only on conversational or trivial turns.';
  const MODE_REFRESH =
      'Orchestration mode is still on. Use the Workflow tool; see its standing consent section.';
  const MODE_EXIT =
      'Orchestration mode is off. The Workflow tool\'s standard opt-in rule applies again.';
  ```

  ```ruby Ruby
  MODE_ENTER =
    "Orchestration mode is on: optimize for the most exhaustive, correct answer rather than " \
    "the fastest one. Use the Workflow tool on every substantive task, sized to the problem's " \
    "natural decomposition rather than the maximum the tool allows. See the Workflow tool's " \
    "description for standing consent, granularity guidance, and quality patterns. Work solo " \
    "only on conversational or trivial turns."
  MODE_REFRESH =
    "Orchestration mode is still on. Use the Workflow tool; see its standing consent section."
  MODE_EXIT =
    "Orchestration mode is off. The Workflow tool's standard opt-in rule applies again."
  ```
</CodeGroup>

## 在工具描述中授予持续同意

Workflow 工具承载着真正的行为契约：选择启用规则、模式开启期间适用的持续同意、用于确定扇出规模的粒度指导，以及模型可以采用的质量模式（验证波次、完整性评审、多阶段排序）。子智能体还会获得一个 `report_findings` 工具，以便它们的结果以结构化 JSON 而非散文形式返回，而 bash 工具则是在本地运行的 Anthropic 定义的 `bash_20250124` 工具。

<CodeGroup>
  ```python Python
  WORKFLOW_TOOL = {
      "name": "Workflow",
      "description": (
          "Orchestrate a multiagent workflow: split a large task into independent subtasks "
          "and run them as parallel agents, then collect their results.\n\n"
          "Opt-in: only use this tool when the user explicitly asks for a workflow, or when a "
          "system message confirms that orchestration mode is on.\n\n"
          "Quality patterns: adversarial verification (a second wave of agents checks the first "
          "wave's findings against the source), a completeness critic (one agent hunts for what "
          "the others missed), and multiphase sequencing (understand, design, implement, and "
          "review as separate workflow calls, reading results between phases). A useful default "
          "is hybrid: scout inline first to discover the work-list, then fan out over it.\n\n"
          "Granularity: scope each subtask to a distinct concern, component, or question rather "
          "than per line or per file section. Scale the count to what the user asked for: a "
          "focused review of a module of a few hundred lines rarely needs more than about ten "
          "subtasks; a broad audit of a large codebase can justify more.\n\n"
          "Standing consent: while a system message confirms orchestration mode is on, that "
          "opt-in is standing. Author and run a workflow for every substantive task by default, "
          "and lean toward verifying findings adversarially. Work solo only on conversational "
          "turns or trivial mechanical edits. When a system message says the mode is off, "
          "revert to the opt-in rule above."
      ),
      "input_schema": {
          "type": "object",
          "properties": {
              "subtasks": {
                  "type": "array",
                  "items": {"type": "string"},
                  "description": "Independent subtask prompts to run as parallel agents",
              }
          },
          "required": ["subtasks"],
      },
  }

  BASH_TOOL = {"type": "bash_20250124", "name": "bash"}

  REPORT_TOOL = {
      "name": "report_findings",
      "description": (
          "Report the final findings for your subtask. Call this exactly once, when you are "
          "done investigating; it ends your task."
      ),
      "input_schema": {
          "type": "object",
          "properties": {
              "summary": {"type": "string", "description": "Two or three sentences of synthesis"},
              "findings": {
                  "type": "array",
                  "items": {
                      "type": "object",
                      "properties": {
                          "claim": {"type": "string", "description": "The finding, one sentence"},
                          "evidence": {
                              "type": "string",
                              "description": "How it was verified (file, line, or command output)",
                          },
                          "severity": {"type": "string", "enum": ["high", "medium", "low", "info"]},
                      },
                      "required": ["claim", "evidence", "severity"],
                  },
              },
          },
          "required": ["summary", "findings"],
      },
  }
  ```

  ```typescript TypeScript
  const WORKFLOW_TOOL: Anthropic.Tool = {
    name: "Workflow",
    description:
      "Orchestrate a multiagent workflow: split a large task into independent subtasks " +
      "and run them as parallel agents, then collect their results.\n\n" +
      "Opt-in: only use this tool when the user explicitly asks for a workflow, or when a " +
      "system message confirms that orchestration mode is on.\n\n" +
      "Quality patterns: adversarial verification (a second wave of agents checks the first " +
      "wave's findings against the source), a completeness critic (one agent hunts for what " +
      "the others missed), and multiphase sequencing (understand, design, implement, and " +
      "review as separate workflow calls, reading results between phases). A useful default " +
      "is hybrid: scout inline first to discover the work-list, then fan out over it.\n\n" +
      "Granularity: scope each subtask to a distinct concern, component, or question rather " +
      "than per line or per file section. Scale the count to what the user asked for: a " +
      "focused review of a module of a few hundred lines rarely needs more than about ten " +
      "subtasks; a broad audit of a large codebase can justify more.\n\n" +
      "Standing consent: while a system message confirms orchestration mode is on, that " +
      "opt-in is standing. Author and run a workflow for every substantive task by default, " +
      "and lean toward verifying findings adversarially. Work solo only on conversational " +
      "turns or trivial mechanical edits. When a system message says the mode is off, " +
      "revert to the opt-in rule above.",
    input_schema: {
      type: "object",
      properties: {
        subtasks: {
          type: "array",
          items: { type: "string" },
          description: "Independent subtask prompts to run as parallel agents",
        },
      },
      required: ["subtasks"],
    },
  };

  const BASH_TOOL: Anthropic.ToolBash20250124 = { type: "bash_20250124", name: "bash" };

  const REPORT_TOOL: Anthropic.Tool = {
    name: "report_findings",
    description:
      "Report the final findings for your subtask. Call this exactly once, when you are " +
      "done investigating; it ends your task.",
    input_schema: {
      type: "object",
      properties: {
        summary: { type: "string", description: "Two or three sentences of synthesis" },
        findings: {
          type: "array",
          items: {
            type: "object",
            properties: {
              claim: { type: "string", description: "The finding, one sentence" },
              evidence: {
                type: "string",
                description: "How it was verified (file, line, or command output)",
              },
              severity: { type: "string", enum: ["high", "medium", "low", "info"] },
            },
            required: ["claim", "evidence", "severity"],
          },
        },
      },
      required: ["summary", "findings"],
    },
  };
  ```

  ```csharp C#
  Tool workflowTool = new()
  {
      Name = "Workflow",
      Description =
          "Orchestrate a multiagent workflow: split a large task into independent subtasks "
          + "and run them as parallel agents, then collect their results.\n\n"
          + "Opt-in: only use this tool when the user explicitly asks for a workflow, or when a "
          + "system message confirms that orchestration mode is on.\n\n"
          + "Quality patterns: adversarial verification (a second wave of agents checks the first "
          + "wave's findings against the source), a completeness critic (one agent hunts for what "
          + "the others missed), and multiphase sequencing (understand, design, implement, and "
          + "review as separate workflow calls, reading results between phases). A useful default "
          + "is hybrid: scout inline first to discover the work-list, then fan out over it.\n\n"
          + "Granularity: scope each subtask to a distinct concern, component, or question rather "
          + "than per line or per file section. Scale the count to what the user asked for: a "
          + "focused review of a module of a few hundred lines rarely needs more than about ten "
          + "subtasks; a broad audit of a large codebase can justify more.\n\n"
          + "Standing consent: while a system message confirms orchestration mode is on, that "
          + "opt-in is standing. Author and run a workflow for every substantive task by default, "
          + "and lean toward verifying findings adversarially. Work solo only on conversational "
          + "turns or trivial mechanical edits. When a system message says the mode is off, "
          + "revert to the opt-in rule above.",
      InputSchema = new InputSchema
      {
          Properties = new Dictionary<string, JsonElement>
          {
              ["subtasks"] = JsonSerializer.SerializeToElement(new
              {
                  type = "array",
                  items = new { type = "string" },
                  description = "Independent subtask prompts to run as parallel agents",
              }),
          },
          Required = ["subtasks"],
      },
  };

  ToolBash20250124 bashTool = new();

  Tool reportTool = new()
  {
      Name = "report_findings",
      Description =
          "Report the final findings for your subtask. Call this exactly once, when you are "
          + "done investigating; it ends your task.",
      InputSchema = new InputSchema
      {
          Properties = new Dictionary<string, JsonElement>
          {
              ["summary"] = JsonSerializer.SerializeToElement(new
              {
                  type = "string",
                  description = "Two or three sentences of synthesis",
              }),
              ["findings"] = JsonSerializer.SerializeToElement(new
              {
                  type = "array",
                  items = new
                  {
                      type = "object",
                      properties = new
                      {
                          claim = new { type = "string", description = "The finding, one sentence" },
                          evidence = new
                          {
                              type = "string",
                              description = "How it was verified (file, line, or command output)",
                          },
                          severity = new { type = "string", @enum = new[] { "high", "medium", "low", "info" } },
                      },
                      required = new[] { "claim", "evidence", "severity" },
                  },
              }),
          },
          Required = ["summary", "findings"],
      },
  };
  ```

  ```go Go
  var workflowTool = anthropic.ToolUnionParam{
  	OfTool: &anthropic.ToolParam{
  		Name: "Workflow",
  		Description: anthropic.String("Orchestrate a multiagent workflow: split a large task into independent subtasks " +
  			"and run them as parallel agents, then collect their results.\n\n" +
  			"Opt-in: only use this tool when the user explicitly asks for a workflow, or when a " +
  			"system message confirms that orchestration mode is on.\n\n" +
  			"Quality patterns: adversarial verification (a second wave of agents checks the first " +
  			"wave's findings against the source), a completeness critic (one agent hunts for what " +
  			"the others missed), and multiphase sequencing (understand, design, implement, and " +
  			"review as separate workflow calls, reading results between phases). A useful default " +
  			"is hybrid: scout inline first to discover the work-list, then fan out over it.\n\n" +
  			"Granularity: scope each subtask to a distinct concern, component, or question rather " +
  			"than per line or per file section. Scale the count to what the user asked for: a " +
  			"focused review of a module of a few hundred lines rarely needs more than about ten " +
  			"subtasks; a broad audit of a large codebase can justify more.\n\n" +
  			"Standing consent: while a system message confirms orchestration mode is on, that " +
  			"opt-in is standing. Author and run a workflow for every substantive task by default, " +
  			"and lean toward verifying findings adversarially. Work solo only on conversational " +
  			"turns or trivial mechanical edits. When a system message says the mode is off, " +
  			"revert to the opt-in rule above."),
  		InputSchema: anthropic.ToolInputSchemaParam{
  			Properties: map[string]any{
  				"subtasks": map[string]any{
  					"type":        "array",
  					"items":       map[string]any{"type": "string"},
  					"description": "Independent subtask prompts to run as parallel agents",
  				},
  			},
  			Required: []string{"subtasks"},
  		},
  	},
  }

  var bashTool = anthropic.ToolUnionParam{
  	OfBashTool20250124: &anthropic.ToolBash20250124Param{},
  }

  var reportTool = anthropic.ToolUnionParam{
  	OfTool: &anthropic.ToolParam{
  		Name: "report_findings",
  		Description: anthropic.String("Report the final findings for your subtask. Call this exactly once, when you are " +
  			"done investigating; it ends your task."),
  		InputSchema: anthropic.ToolInputSchemaParam{
  			Properties: map[string]any{
  				"summary": map[string]any{"type": "string", "description": "Two or three sentences of synthesis"},
  				"findings": map[string]any{
  					"type": "array",
  					"items": map[string]any{
  						"type": "object",
  						"properties": map[string]any{
  							"claim": map[string]any{"type": "string", "description": "The finding, one sentence"},
  							"evidence": map[string]any{
  								"type":        "string",
  								"description": "How it was verified (file, line, or command output)",
  							},
  							"severity": map[string]any{"type": "string", "enum": []string{"high", "medium", "low", "info"}},
  						},
  						"required": []string{"claim", "evidence", "severity"},
  					},
  				},
  			},
  			Required: []string{"summary", "findings"},
  		},
  	},
  }

  ```

  ```java Java
  static final Tool WORKFLOW_TOOL = Tool.builder()
          .name("Workflow")
          .description("Orchestrate a multiagent workflow: split a large task into independent subtasks "
                  + "and run them as parallel agents, then collect their results.\n\n"
                  + "Opt-in: only use this tool when the user explicitly asks for a workflow, or when a "
                  + "system message confirms that orchestration mode is on.\n\n"
                  + "Quality patterns: adversarial verification (a second wave of agents checks the first "
                  + "wave's findings against the source), a completeness critic (one agent hunts for what "
                  + "the others missed), and multiphase sequencing (understand, design, implement, and "
                  + "review as separate workflow calls, reading results between phases). A useful default "
                  + "is hybrid: scout inline first to discover the work-list, then fan out over it.\n\n"
                  + "Granularity: scope each subtask to a distinct concern, component, or question rather "
                  + "than per line or per file section. Scale the count to what the user asked for: a "
                  + "focused review of a module of a few hundred lines rarely needs more than about ten "
                  + "subtasks; a broad audit of a large codebase can justify more.\n\n"
                  + "Standing consent: while a system message confirms orchestration mode is on, that "
                  + "opt-in is standing. Author and run a workflow for every substantive task by default, "
                  + "and lean toward verifying findings adversarially. Work solo only on conversational "
                  + "turns or trivial mechanical edits. When a system message says the mode is off, "
                  + "revert to the opt-in rule above.")
          .inputSchema(Tool.InputSchema.builder()
                  .properties(JsonValue.from(Map.of(
                          "subtasks", Map.of(
                                  "type", "array",
                                  "items", Map.of("type", "string"),
                                  "description", "Independent subtask prompts to run as parallel agents"))))
                  .putAdditionalProperty("required", JsonValue.from(List.of("subtasks")))
                  .build())
          .build();

  static final ToolBash20250124 BASH_TOOL = ToolBash20250124.builder().build();

  static final Tool REPORT_TOOL = Tool.builder()
          .name("report_findings")
          .description("Report the final findings for your subtask. Call this exactly once, when you are "
                  + "done investigating; it ends your task.")
          .inputSchema(Tool.InputSchema.builder()
                  .properties(JsonValue.from(Map.of(
                          "summary", Map.of("type", "string", "description", "Two or three sentences of synthesis"),
                          "findings", Map.of(
                                  "type", "array",
                                  "items", Map.of(
                                          "type", "object",
                                          "properties", Map.of(
                                                  "claim", Map.of(
                                                          "type", "string",
                                                          "description", "The finding, one sentence"),
                                                  "evidence", Map.of(
                                                          "type", "string",
                                                          "description", "How it was verified (file, line, or command output)"),
                                                  "severity", Map.of(
                                                          "type", "string",
                                                          "enum", List.of("high", "medium", "low", "info"))),
                                          "required", List.of("claim", "evidence", "severity"))))))
                  .putAdditionalProperty("required", JsonValue.from(List.of("summary", "findings")))
                  .build())
          .build();
  ```

  ```php PHP
  const WORKFLOW_TOOL = [
      'name' => 'Workflow',
      'description' =>
          'Orchestrate a multiagent workflow: split a large task into independent subtasks '
          . "and run them as parallel agents, then collect their results.\n\n"
          . 'Opt-in: only use this tool when the user explicitly asks for a workflow, or when a '
          . "system message confirms that orchestration mode is on.\n\n"
          . 'Quality patterns: adversarial verification (a second wave of agents checks the first '
          . 'wave\'s findings against the source), a completeness critic (one agent hunts for what '
          . 'the others missed), and multiphase sequencing (understand, design, implement, and '
          . 'review as separate workflow calls, reading results between phases). A useful default '
          . "is hybrid: scout inline first to discover the work-list, then fan out over it.\n\n"
          . 'Granularity: scope each subtask to a distinct concern, component, or question rather '
          . 'than per line or per file section. Scale the count to what the user asked for: a '
          . 'focused review of a module of a few hundred lines rarely needs more than about ten '
          . "subtasks; a broad audit of a large codebase can justify more.\n\n"
          . 'Standing consent: while a system message confirms orchestration mode is on, that '
          . 'opt-in is standing. Author and run a workflow for every substantive task by default, '
          . 'and lean toward verifying findings adversarially. Work solo only on conversational '
          . 'turns or trivial mechanical edits. When a system message says the mode is off, '
          . 'revert to the opt-in rule above.',
      'input_schema' => [
          'type' => 'object',
          'properties' => [
              'subtasks' => [
                  'type' => 'array',
                  'items' => ['type' => 'string'],
                  'description' => 'Independent subtask prompts to run as parallel agents',
              ],
          ],
          'required' => ['subtasks'],
      ],
  ];

  const BASH_TOOL = ['type' => 'bash_20250124', 'name' => 'bash'];

  const REPORT_TOOL = [
      'name' => 'report_findings',
      'description' =>
          'Report the final findings for your subtask. Call this exactly once, when you are '
          . 'done investigating; it ends your task.',
      'input_schema' => [
          'type' => 'object',
          'properties' => [
              'summary' => ['type' => 'string', 'description' => 'Two or three sentences of synthesis'],
              'findings' => [
                  'type' => 'array',
                  'items' => [
                      'type' => 'object',
                      'properties' => [
                          'claim' => ['type' => 'string', 'description' => 'The finding, one sentence'],
                          'evidence' => [
                              'type' => 'string',
                              'description' => 'How it was verified (file, line, or command output)',
                          ],
                          'severity' => ['type' => 'string', 'enum' => ['high', 'medium', 'low', 'info']],
                      ],
                      'required' => ['claim', 'evidence', 'severity'],
                  ],
              ],
          ],
          'required' => ['summary', 'findings'],
      ],
  ];
  ```

  ```ruby Ruby
  WORKFLOW_TOOL = {
    name: "Workflow",
    description:
      "Orchestrate a multiagent workflow: split a large task into independent subtasks " \
      "and run them as parallel agents, then collect their results.\n\n" \
      "Opt-in: only use this tool when the user explicitly asks for a workflow, or when a " \
      "system message confirms that orchestration mode is on.\n\n" \
      "Quality patterns: adversarial verification (a second wave of agents checks the first " \
      "wave's findings against the source), a completeness critic (one agent hunts for what " \
      "the others missed), and multiphase sequencing (understand, design, implement, and " \
      "review as separate workflow calls, reading results between phases). A useful default " \
      "is hybrid: scout inline first to discover the work-list, then fan out over it.\n\n" \
      "Granularity: scope each subtask to a distinct concern, component, or question rather " \
      "than per line or per file section. Scale the count to what the user asked for: a " \
      "focused review of a module of a few hundred lines rarely needs more than about ten " \
      "subtasks; a broad audit of a large codebase can justify more.\n\n" \
      "Standing consent: while a system message confirms orchestration mode is on, that " \
      "opt-in is standing. Author and run a workflow for every substantive task by default, " \
      "and lean toward verifying findings adversarially. Work solo only on conversational " \
      "turns or trivial mechanical edits. When a system message says the mode is off, " \
      "revert to the opt-in rule above.",
    input_schema: {
      type: "object",
      properties: {
        subtasks: {
          type: "array",
          items: {type: "string"},
          description: "Independent subtask prompts to run as parallel agents"
        }
      },
      required: ["subtasks"]
    }
  }.freeze

  BASH_TOOL = {type: "bash_20250124", name: "bash"}.freeze

  REPORT_TOOL = {
    name: "report_findings",
    description:
      "Report the final findings for your subtask. Call this exactly once, when you are " \
      "done investigating; it ends your task.",
    input_schema: {
      type: "object",
      properties: {
        summary: {type: "string", description: "Two or three sentences of synthesis"},
        findings: {
          type: "array",
          items: {
            type: "object",
            properties: {
              claim: {type: "string", description: "The finding, one sentence"},
              evidence: {
                type: "string",
                description: "How it was verified (file, line, or command output)"
              },
              severity: {type: "string", enum: ["high", "medium", "low", "info"]}
            },
            required: ["claim", "evidence", "severity"]
          }
        }
      },
      required: ["summary", "findings"]
    }
  }.freeze
  ```
</CodeGroup>

## 在本地运行 bash 工具

bash 处理程序以超时方式运行所请求的命令，捕获合并的 stdout 和 stderr，并截断结果，以免失控的命令淹没 "context window"（上下文窗口）。命令在您启动示例的目录中运行，因此要将其指向某个项目，就需要在该项目目录中启动它；当设置了 `DOC_TEST_MODE` 时，测试框架会改为给 bash 提供一个小型的临时夹具目录，并在退出时删除。这里没有沙箱：命令以启动示例的进程的权限运行。为清晰起见，本示例在全新的子 shell 中运行每次调用，而不是维护 `bash_20250124` 契约所描述的持久会话；生产环境的智能体应使用长期存活的 shell 来支撑该工具，以便工作目录、环境和 `restart` 操作的行为与文档一致。

<CodeGroup>
  ```python Python
  # 在示例启动的位置运行 bash。在 DOC_TEST_MODE 下，文档测试框架
  # 会将其改为指向一个一次性的 fixture 目录，并在退出时删除。
  if DOC_TEST_MODE:
      WORK_DIR = tempfile.mkdtemp(prefix="orchestration-")
      atexit.register(shutil.rmtree, WORK_DIR, ignore_errors=True)
      with open(os.path.join(WORK_DIR, "sample.py"), "w") as fixture:
          fixture.write(
              "def fib(n):\n"
              "    return n if n < 2 else fib(n - 1) + fib(n - 2)\n\n"
              "print(fib(10))\n"
          )
  else:
      WORK_DIR = os.getcwd()


  def run_bash(command: str) -> tuple[str, bool]:
      """Run a shell command and return (output, is_error). No sandbox: example code only."""
      print(f"[bash] {command}", file=sys.stderr)
      try:
          proc = subprocess.run(
              ["bash", "-c", command],
              cwd=WORK_DIR,
              capture_output=True,
              text=True,
              errors="replace",
              timeout=BASH_TIMEOUT_SECONDS,
          )
      except subprocess.TimeoutExpired:
          return f"command timed out after {BASH_TIMEOUT_SECONDS}s", True
      output = (proc.stdout + proc.stderr).strip() or "(no output)"
      if len(output) > TOOL_RESULT_MAX_CHARS:
          output = output[:TOOL_RESULT_MAX_CHARS] + f"\n(truncated at {TOOL_RESULT_MAX_CHARS} chars)"
      if proc.returncode != 0:
          output = f"(exit code {proc.returncode})\n{output}"
      return output, proc.returncode != 0


  def handle_bash_block(block) -> tuple[str, bool]:
      if block.input.get("restart") is True:
          return "Shell restarted.", False
      command = block.input.get("command")
      if not isinstance(command, str) or not command:
          return "bash error: no command was provided.", True
      return run_bash(command)
  ```

  ```typescript TypeScript
  const execShell = promisify(exec);

  // 在示例启动的位置运行 bash。在 DOC_TEST_MODE 下，文档测试框架
  // 会将其指向一个一次性的 fixture 目录，并在退出时删除。
  const WORK_DIR = DOC_TEST_MODE
    ? await mkdtemp(join(tmpdir(), "orchestration-"))
    : process.cwd();
  if (DOC_TEST_MODE) {
    await writeFile(
      join(WORK_DIR, "sample.py"),
      "def fib(n):\n" +
        "    return n if n < 2 else fib(n - 1) + fib(n - 2)\n\n" +
        "print(fib(10))\n",
    );
    process.on("exit", () => rmSync(WORK_DIR, { recursive: true, force: true }));
  }

  // 运行一条 shell 命令并返回其输出。无沙箱：仅为示例代码。
  async function runBash(command: string): Promise<{ output: string; isError: boolean }> {
    console.error(`[bash] ${command}`);
    let stdout = "";
    let stderr = "";
    let exitCode = 0;
    try {
      ({ stdout, stderr } = await execShell(command, {
        shell: "/bin/bash",
        cwd: WORK_DIR,
        timeout: BASH_TIMEOUT_SECONDS * 1000,
        maxBuffer: 16 * 1024 * 1024,
      }));
    } catch (error) {
      const failure = error as {
        stdout?: string;
        stderr?: string;
        code?: number | string;
        killed?: boolean;
      };
      if (failure.killed && failure.code !== "ERR_CHILD_PROCESS_STDIO_MAXBUFFER") {
        return { output: `command timed out after ${BASH_TIMEOUT_SECONDS}s`, isError: true };
      }
      stdout = failure.stdout ?? "";
      stderr = failure.stderr ?? "";
      exitCode = typeof failure.code === "number" ? failure.code : 1;
    }
    let output = (stdout + stderr).trim() || "(no output)";
    const codePoints = [...output];
    if (codePoints.length > TOOL_RESULT_MAX_CHARS) {
      output =
        codePoints.slice(0, TOOL_RESULT_MAX_CHARS).join("") +
        `\n(truncated at ${TOOL_RESULT_MAX_CHARS} chars)`;
    }
    if (exitCode !== 0) {
      output = `(exit code ${exitCode})\n${output}`;
    }
    return { output, isError: exitCode !== 0 };
  }

  async function handleBashBlock(
    block: Anthropic.ToolUseBlock,
  ): Promise<{ output: string; isError: boolean }> {
    const input = block.input as { command?: string; restart?: boolean };
    if (input.restart === true) {
      return { output: "Shell restarted.", isError: false };
    }
    if (!input.command) {
      return { output: "bash error: no command was provided.", isError: true };
    }
    return runBash(input.command);
  }
  ```

  ```csharp C#
  // 在示例启动的位置运行 bash。在 DOC_TEST_MODE 下，文档测试框架
  // 会将其指向一个临时的 fixture 目录，退出时删除。
  var workDir = Environment.CurrentDirectory;
  if (docTestMode)
  {
      workDir = Directory.CreateTempSubdirectory("orchestration-").FullName;
      File.WriteAllText(Path.Combine(workDir, "sample.py"),
          "def fib(n):\n" +
          "    return n if n < 2 else fib(n - 1) + fib(n - 2)\n\n" +
          "print(fib(10))\n");
      var fixtureDir = workDir;
      AppDomain.CurrentDomain.ProcessExit += (_, _) =>
      {
          try { Directory.Delete(fixtureDir, recursive: true); }
          catch { /* Best-effort cleanup; the OS tmp sweeper handles leftovers. */ }
      };
  }

  // 运行 shell 命令并返回其输出及错误标志。无沙箱：仅为示例代码。
  async Task<(string Output, bool IsError)> RunBash(string command)
  {
      Console.Error.WriteLine($"[bash] {command}");
      using var process = Process.Start(new ProcessStartInfo("bash")
      {
          ArgumentList = { "-c", command },
          WorkingDirectory = workDir,
          RedirectStandardOutput = true,
          RedirectStandardError = true,
      });
      if (process is null)
      {
          return ("bash error: the shell process failed to start.", true);
      }
      var stdoutTask = process.StandardOutput.ReadToEndAsync();
      var stderrTask = process.StandardError.ReadToEndAsync();
      using var timeout = new CancellationTokenSource(TimeSpan.FromSeconds(bashTimeoutSeconds));
      try
      {
          await process.WaitForExitAsync(timeout.Token);
      }
      catch (OperationCanceledException)
      {
          process.Kill(entireProcessTree: true);
          // 在进程被释放之前，让读取任务先完成。
          try
          {
              await Task.WhenAll(stdoutTask, stderrTask);
          }
          catch
          {
              // 超时时输出会被丢弃，因此读取失败也一并忽略。
          }
          return ($"command timed out after {bashTimeoutSeconds}s", true);
      }
      var output = (await stdoutTask + await stderrTask).Trim();
      if (output.Length == 0)
      {
          output = "(no output)";
      }
      if (output.Length > toolResultMaxChars)
      {
          output = output[..toolResultMaxChars] + $"\n(truncated at {toolResultMaxChars} chars)";
      }
      if (process.ExitCode != 0)
      {
          output = $"(exit code {process.ExitCode})\n{output}";
      }
      return (output, process.ExitCode != 0);
  }

  // 执行模型请求的一次 bash 工具调用。
  async Task<(string Output, bool IsError)> HandleBashBlock(ToolUseBlock block)
  {
      if (block.Input.TryGetValue("restart", out var restart) && restart.ValueKind == JsonValueKind.True)
      {
          return ("Shell restarted.", false);
      }
      var command = block.Input.TryGetValue("command", out var rawCommand) && rawCommand.ValueKind == JsonValueKind.String
          ? rawCommand.GetString()!
          : "";
      if (command.Length == 0)
      {
          return ("bash error: no command was provided.", true);
      }
      return await RunBash(command);
  }
  ```

  ```go Go
  // 在示例启动的位置运行 bash。在 DOC_TEST_MODE 下，文档测试框架
  // 会将其指向一个临时的 fixture 目录，退出时删除。
  var workDir = func() string {
  	if !docTestMode {
  		dir, err := os.Getwd()
  		if err != nil {
  			log.Fatal(err)
  		}
  		return dir
  	}
  	dir, err := os.MkdirTemp("", "orchestration-")
  	if err != nil {
  		log.Fatal(err)
  	}
  	fixture := "def fib(n):\n" +
  		"    return n if n < 2 else fib(n - 1) + fib(n - 2)\n\n" +
  		"print(fib(10))\n"
  	if err := os.WriteFile(filepath.Join(dir, "sample.py"), []byte(fixture), 0o644); err != nil {
  		log.Fatal(err)
  	}
  	return dir
  }()

  // runBash 运行一条 shell 命令并返回其输出及错误标志。
  // 无沙箱：仅为示例代码。
  func runBash(ctx context.Context, command string) (string, bool) {
  	fmt.Fprintf(os.Stderr, "[bash] %s\n", command)
  	ctx, cancel := context.WithTimeout(ctx, bashTimeoutSeconds*time.Second)
  	defer cancel()
  	cmd := exec.CommandContext(ctx, "bash", "-c", command)
  	cmd.Dir = workDir
  	combined, err := cmd.CombinedOutput()
  	if errors.Is(ctx.Err(), context.DeadlineExceeded) {
  		return fmt.Sprintf("command timed out after %ds", bashTimeoutSeconds), true
  	}
  	output := strings.TrimSpace(string(combined))
  	if output == "" {
  		output = "(no output)"
  	}
  	if runes := []rune(output); len(runes) > toolResultMaxChars {
  		output = string(runes[:toolResultMaxChars]) + fmt.Sprintf("\n(truncated at %d chars)", toolResultMaxChars)
  	}
  	if err == nil {
  		return output, false
  	}
  	var exitErr *exec.ExitError
  	if errors.As(err, &exitErr) {
  		return fmt.Sprintf("(exit code %d)\n%s", exitErr.ExitCode(), output), true
  	}
  	return fmt.Sprintf("(%s)\n%s", err, output), true
  }

  // handleBashBlock 执行模型请求的一次 bash 工具调用。
  func handleBashBlock(ctx context.Context, block anthropic.ToolUseBlock) (string, bool) {
  	var input struct {
  		Command string `json:"command"`
  		Restart bool   `json:"restart"`
  	}
  	if err := json.Unmarshal(block.Input, &input); err != nil {
  		return fmt.Sprintf("bash error: could not parse input: %s", err), true
  	}
  	if input.Restart {
  		return "Shell restarted.", false
  	}
  	if input.Command == "" {
  		return "bash error: no command was provided.", true
  	}
  	return runBash(ctx, input.Command)
  }

  ```

  ```java Java
  record ToolOutput(String output, boolean isError) {}

  // 在示例启动的位置运行 bash。在 DOC_TEST_MODE 下，文档测试框架
  // 会将其指向一个临时 fixture 目录，退出时删除。
  static final Path WORK_DIR = createWorkDir();

  static Path createWorkDir() {
      if (!DOC_TEST_MODE) {
          return Path.of(System.getProperty("user.dir"));
      }
      try {
          var dir = Files.createTempDirectory("orchestration-");
          Files.writeString(dir.resolve("sample.py"), """
                  def fib(n):
                      return n if n < 2 else fib(n - 1) + fib(n - 2)

                  print(fib(10))
                  """);
          Runtime.getRuntime().addShutdownHook(new Thread(() -> {
              try (var paths = Files.walk(dir)) {
                  paths.sorted(Comparator.reverseOrder()).forEach(p -> {
                      try { Files.deleteIfExists(p); } catch (IOException ignored) {}
                  });
              } catch (IOException ignored) {
                  // 尽力清理；剩余文件由操作系统的 tmp 清理程序处理。
              }
          }));
          return dir;
      } catch (IOException error) {
          throw new UncheckedIOException(error);
      }
  }

  // 运行 shell 命令并返回其输出及错误标志。无沙箱：仅作示例代码。
  ToolOutput runBash(String command) throws InterruptedException {
      System.err.println("[bash] " + command);
      Process process;
      try {
          process = new ProcessBuilder("bash", "-c", command)
                  .directory(WORK_DIR.toFile())
                  .redirectErrorStream(true)
                  .start();
      } catch (IOException error) {
          return new ToolOutput("(" + error + ")", true);
      }
      // 在另一线程上读取 stdout，以免管道填满导致下方的超时等待卡住。
      CompletableFuture<String> outputReader = CompletableFuture.supplyAsync(() -> {
          try (var stdout = process.getInputStream()) {
              return new String(stdout.readAllBytes(), StandardCharsets.UTF_8);
          } catch (IOException error) {
              return "";
          }
      });
      if (!process.waitFor(BASH_TIMEOUT_SECONDS, TimeUnit.SECONDS)) {
          process.destroyForcibly();
          outputReader.cancel(true);
          return new ToolOutput("command timed out after " + BASH_TIMEOUT_SECONDS + "s", true);
      }
      String output = outputReader.join().trim();
      if (output.isEmpty()) {
          output = "(no output)";
      }
      if (output.length() > TOOL_RESULT_MAX_CHARS) {
          output = output.substring(0, TOOL_RESULT_MAX_CHARS)
                  + "\n(truncated at " + TOOL_RESULT_MAX_CHARS + " chars)";
      }
      int exitCode = process.exitValue();
      if (exitCode != 0) {
          return new ToolOutput("(exit code " + exitCode + ")\n" + output, true);
      }
      return new ToolOutput(output, false);
  }

  // 执行模型请求的一次 bash 工具调用。
  ToolOutput handleBashBlock(ToolUseBlock block) throws InterruptedException {
      Map<String, JsonValue> input = (Map<String, JsonValue>) block._input().asObject().orElse(Map.of());
      JsonValue restart = input.getOrDefault("restart", JsonValue.from(false));
      if (Boolean.TRUE.equals(restart.asBoolean().orElse(false))) {
          return new ToolOutput("Shell restarted.", false);
      }
      JsonValue raw = input.get("command");
      String command = raw != null && raw.asString().isPresent() ? raw.asStringOrThrow() : "";
      if (command.isEmpty()) {
          return new ToolOutput("bash error: no command was provided.", true);
      }
      return runBash(command);
  }
  ```

  ```php PHP
  // 在示例启动的位置运行 bash。在 DOC_TEST_MODE 下，文档测试框架
  // 会将其改为指向一个一次性的 fixture 目录，并在退出时删除。
  if (DOC_TEST_MODE) {
      $workDir = sys_get_temp_dir() . '/orchestration-' . bin2hex(random_bytes(8));
      if (!mkdir($workDir, 0700)) {
          throw new RuntimeException("could not create working directory {$workDir}");
      }
      file_put_contents(
          $workDir . '/sample.py',
          "def fib(n):\n"
          . "    return n if n < 2 else fib(n - 1) + fib(n - 2)\n\n"
          . "print(fib(10))\n",
      );
      register_shutdown_function(function () use ($workDir): void {
          foreach (glob($workDir . '/*') ?: [] as $entry) {
              @unlink($entry);
          }
          @rmdir($workDir);
      });
  } else {
      $workDir = getcwd() ?: '.';
  }
  define('WORK_DIR', $workDir);

  /**
   * Run a shell command and return [output, isError]. The coreutils timeout command
   * enforces the time limit. No sandbox: example code only.
   */
  function runBash(string $command): array
  {
      fwrite(STDERR, "[bash] {$command}\n");
      // 需要 GNU coreutils 的 'timeout'。在 macOS 上：brew install coreutils，或替换为 gtimeout。
      exec(
          'cd ' . escapeshellarg(WORK_DIR) . ' && timeout ' . BASH_TIMEOUT_SECONDS
              . ' bash -c ' . escapeshellarg($command) . ' 2>&1',
          $outputLines,
          $exitCode,
      );
      if ($exitCode === 124) {
          return ['command timed out after ' . BASH_TIMEOUT_SECONDS . 's', true];
      }
      $output = trim(implode("\n", $outputLines));
      if ($output === '') {
          $output = '(no output)';
      }
      if (mb_strlen($output) > TOOL_RESULT_MAX_CHARS) {
          $output = mb_substr($output, 0, TOOL_RESULT_MAX_CHARS)
              . "\n(truncated at " . TOOL_RESULT_MAX_CHARS . ' chars)';
      }
      if ($exitCode !== 0) {
          $output = "(exit code {$exitCode})\n{$output}";
      }
      return [$output, $exitCode !== 0];
  }

  /** Execute one bash tool call requested by the model. */
  function handleBashBlock(ToolUseBlock $block): array
  {
      if (($block->input['restart'] ?? null) === true) {
          return ['Shell restarted.', false];
      }
      $command = $block->input['command'] ?? '';
      if (!is_string($command) || $command === '') {
          return ['bash error: no command was provided.', true];
      }
      return runBash($command);
  }
  ```

  ```ruby Ruby
  # 在示例启动的位置运行 bash。在 DOC_TEST_MODE 下，文档测试框架
  # 会将其指向一个一次性的 fixture 目录，并在退出时删除。
  WORK_DIR =
    if DOC_TEST_MODE
      Dir.mktmpdir("orchestration-").tap do |dir|
        File.write(File.join(dir, "sample.py"), <<~PYTHON)
          def fib(n):
              return n if n < 2 else fib(n - 1) + fib(n - 2)

          print(fib(10))
        PYTHON
        at_exit { FileUtils.remove_entry(dir, true) }
      end
    else
      Dir.pwd
    end

  # 工具输入可能是 Hash，也可能是来自流式传输累加器的原始 JSON 字符串；
  # 将这两种形式统一规范化为以字符串为键的 Hash。
  def parse_tool_input(raw)
    return raw.transform_keys(&:to_s) if raw.is_a?(Hash)
    parsed = JSON.parse(raw.to_s) rescue nil
    parsed.is_a?(Hash) ? parsed : {}
  end

  # 运行 shell 命令并返回 [output, is_error]。没有沙箱：仅为示例代码。
  def run_bash(command)
    warn "[bash] #{command}"
    begin
      stdin, stdout_and_stderr, wait_thr = Open3.popen2e("bash", "-c", command, pgroup: true, chdir: WORK_DIR)
      stdin.close
      reader = Thread.new { stdout_and_stderr.read.scrub }
      # 使用单调时钟截止时间来强制执行时间限制，这样超时的命令会被
      # 终止，而不是留在后台继续运行。
      deadline = Process.clock_gettime(Process::CLOCK_MONOTONIC) + BASH_TIMEOUT_SECONDS
      until wait_thr.join(0.1)
        next if Process.clock_gettime(Process::CLOCK_MONOTONIC) < deadline

        begin
          Process.kill("-TERM", wait_thr.pid)
        rescue Errno::ESRCH
        end
        unless wait_thr.join(2)
          begin
            Process.kill("-KILL", wait_thr.pid)
          rescue Errno::ESRCH
          end
        end
        wait_thr.join(5)
        reader.join(1) || reader.kill
        stdout_and_stderr.close rescue nil
        return ["command timed out after #{BASH_TIMEOUT_SECONDS}s", true]
      end
      status = wait_thr.value
      output = reader.value.strip
      stdout_and_stderr.close
      output = "(no output)" if output.empty?
      if output.length > TOOL_RESULT_MAX_CHARS
        output = "#{output[0, TOOL_RESULT_MAX_CHARS]}\n(truncated at #{TOOL_RESULT_MAX_CHARS} chars)"
      end
      output = "(exit code #{status.exitstatus})\n#{output}" unless status.success?
      [output, !status.success?]
    rescue Errno::ENOENT => e
      return ["bash error: #{e.message}", true]
    end
  end

  # 执行模型请求的一次 bash 工具调用。
  def handle_bash_block(block)
    input = parse_tool_input(block.input)
    return ["Shell restarted.", false] if input["restart"] == true

    command = input["command"]
    return ["bash error: no command was provided.", true] unless command.is_a?(String) && !command.empty?

    run_bash(command)
  end

  # 将响应内容转换为请求形式的参数。流式传输累加器
  # 以原始 JSON 字符串返回 tool_use 输入，并包含仅在响应中出现的字段，
  # 因此在回传之前，需将每个块重塑为请求的 schema。
  def assistant_content_param(content)
    content.map do |block|
      case block.type
      when :tool_use
        input = parse_tool_input(block.input)
        {type: "tool_use", id: block.id, name: block.name, input: input}
      when :text
        {type: "text", text: block.text}
      when :thinking
        {type: "thinking", thinking: block.thinking, signature: block.signature}
      when :redacted_thinking then {type: "redacted_thinking", data: block.data}
      else
        block.to_h
      end
    end
  end
  ```
</CodeGroup>

## 运行一个子智能体

每个工作流子任务都会成为一个带有 bash 工具的独立小型智能体循环，以与主循环相同的努力级别运行。每个请求的超时限制了每次 API 调用的时长，因此连接断开只会使一个子智能体降级，而不会使整个运行停滞。

<CodeGroup>
  ```python Python
  def run_subagent(model: str, prompt: str) -> str:
      """One subagent: a small nested agent loop with the bash tool plus report_findings.
      Subagents inherit the main loop's effort level."""
      subagent_system = (
          "You are one agent in a larger parallel fan-out, assigned a single subtask. "
          "Investigate it directly, using bash to check facts rather than guessing, and finish "
          "by calling report_findings exactly once. Return findings, not narration."
      )
      messages = [{"role": "user", "content": prompt}]
      for _ in range(MAX_SUBAGENT_TURNS):
          with client.messages.stream(
              model=model,
              max_tokens=64000,
              system=subagent_system,
              output_config={"effort": EFFORT},
              tools=[BASH_TOOL, REPORT_TOOL],
              messages=messages,
              timeout=REQUEST_TIMEOUT_SECONDS,
          ) as stream:
              response = stream.get_final_message()
          messages.append({"role": "assistant", "content": response.content})
          if response.stop_reason == "pause_turn":
              continue
          if response.stop_reason != "tool_use":
              text = "".join(block.text for block in response.content if block.type == "text")
              if response.stop_reason == "max_tokens":
                  text += "\n\n(warning: subagent response was truncated at max_tokens)"
              return text
          tool_results = []
          report = None
          for block in response.content:
              if block.type != "tool_use":
                  continue
              if block.name == "report_findings":
                  report = json.dumps(block.input, indent=2)
                  output, is_error = "Findings recorded.", False
              elif block.name == "bash":
                  output, is_error = handle_bash_block(block)
              else:
                  output, is_error = f"unknown tool: {block.name}", True
              tool_results.append(
                  {
                      "type": "tool_result",
                      "tool_use_id": block.id,
                      "content": output,
                      "is_error": is_error,
                  }
              )
          if report is not None:
              return report
          messages.append({"role": "user", "content": tool_results})
      return "(subagent hit the turn limit before finishing)"
  ```

  ```typescript TypeScript
  // 一个子代理：一个小型嵌套代理循环，带有 bash 工具和 report_findings。
  // 子代理继承主循环的 effort 级别。
  async function runSubagent(model: string, prompt: string): Promise<string> {
    const subagentSystem =
      "You are one agent in a larger parallel fan-out, assigned a single subtask. " +
      "Investigate it directly, using bash to check facts rather than guessing, and finish " +
      "by calling report_findings exactly once. Return findings, not narration.";
    const messages: Anthropic.MessageParam[] = [{ role: "user", content: prompt }];
    for (let turn = 0; turn < MAX_SUBAGENT_TURNS; turn++) {
      const response = await client.messages
        .stream(
          {
            model,
            max_tokens: 64000,
            system: subagentSystem,
            output_config: { effort: EFFORT },
            tools: [BASH_TOOL, REPORT_TOOL],
            messages,
          },
          { signal: AbortSignal.timeout(REQUEST_TIMEOUT_SECONDS * 1000) },
        )
        .finalMessage();
      messages.push({ role: "assistant", content: response.content });
      if (response.stop_reason === "pause_turn") {
        continue;
      }
      if (response.stop_reason !== "tool_use") {
        let text = response.content
          .filter((block): block is Anthropic.TextBlock => block.type === "text")
          .map((block) => block.text)
          .join("");
        if (response.stop_reason === "max_tokens") {
          text += "\n\n(warning: subagent response was truncated at max_tokens)";
        }
        return text;
      }
      const toolResults: Anthropic.ToolResultBlockParam[] = [];
      let report: string | null = null;
      for (const block of response.content) {
        if (block.type !== "tool_use") {
          continue;
        }
        let output: string;
        let isError: boolean;
        if (block.name === "report_findings") {
          report = JSON.stringify(block.input, null, 2);
          output = "Findings recorded.";
          isError = false;
        } else if (block.name === "bash") {
          ({ output, isError } = await handleBashBlock(block));
        } else {
          output = `unknown tool: ${block.name}`;
          isError = true;
        }
        toolResults.push({
          type: "tool_result",
          tool_use_id: block.id,
          content: output,
          is_error: isError,
        });
      }
      if (report !== null) {
        return report;
      }
      messages.push({ role: "user", content: toolResults });
    }
    return "(subagent hit the turn limit before finishing)";
  }
  ```

  ```csharp C#
  // 一个子代理：一个小型嵌套代理循环，带有 bash 工具和 report_findings。
  // 子代理继承主循环的 effort 级别。
  async Task<string> RunSubagent(string prompt)
  {
      const string subagentSystem =
          "You are one agent in a larger parallel fan-out, assigned a single subtask. "
          + "Investigate it directly, using bash to check facts rather than guessing, and finish "
          + "by calling report_findings exactly once. Return findings, not narration.";
      List<MessageParam> messages = [new() { Role = Role.User, Content = prompt }];
      for (var turn = 0; turn < maxSubagentTurns; turn++)
      {
          using var deadline = new CancellationTokenSource(TimeSpan.FromSeconds(requestTimeoutSeconds));
          var response = await client.Messages.Create(new MessageCreateParams
          {
              Model = model,
              MaxTokens = requestMaxTokens,
              System = subagentSystem,
              OutputConfig = new OutputConfig { Effort = effort },
              Tools = [bashTool, reportTool],
              Messages = messages,
          }, cancellationToken: deadline.Token);
          messages.Add(new()
          {
              Role = Role.Assistant,
              Content = response.Content.Select(block => new ContentBlockParam(block.Json)).ToList(),
          });
          if (response.StopReason == StopReason.PauseTurn)
          {
              continue;
          }
          if (response.StopReason != StopReason.ToolUse)
          {
              var text = string.Concat(
                  response.Content.Select(block => block.TryPickText(out var textBlock) ? textBlock.Text : ""));
              if (response.StopReason == StopReason.MaxTokens)
              {
                  text += "\n\n(warning: subagent response was truncated at max_tokens)";
              }
              return text;
          }
          List<ContentBlockParam> toolResults = [];
          string? report = null;
          foreach (var block in response.Content)
          {
              if (!block.TryPickToolUse(out var toolUse))
              {
                  continue;
              }
              string output;
              bool isError;
              if (toolUse.Name == "report_findings")
              {
                  report = JsonSerializer.Serialize(
                      toolUse.Input, new JsonSerializerOptions { WriteIndented = true });
                  output = "Findings recorded.";
                  isError = false;
              }
              else if (toolUse.Name == "bash")
              {
                  (output, isError) = await HandleBashBlock(toolUse);
              }
              else
              {
                  output = $"unknown tool: {toolUse.Name}";
                  isError = true;
              }
              toolResults.Add(new ToolResultBlockParam(toolUse.ID) { Content = output, IsError = isError });
          }
          if (report is not null)
          {
              return report;
          }
          messages.Add(new() { Role = Role.User, Content = toolResults });
      }
      return "(subagent hit the turn limit before finishing)";
  }
  ```

  ```go Go
  // runSubagent 运行一个子代理：一个带有 bash 工具和 report_findings 的
  // 小型嵌套代理循环。子代理继承主循环的 effort 级别。
  func runSubagent(ctx context.Context, model string, prompt string) (string, error) {
  	subagentSystem := "You are one agent in a larger parallel fan-out, assigned a single subtask. " +
  		"Investigate it directly, using bash to check facts rather than guessing, and finish " +
  		"by calling report_findings exactly once. Return findings, not narration."
  	messages := []anthropic.MessageParam{anthropic.NewUserMessage(anthropic.NewTextBlock(prompt))}
  	for range maxSubagentTurns {
  		var response anthropic.Message
  		err := func() error {
  			ctx, cancel := context.WithTimeout(ctx, requestTimeoutSeconds*time.Second)
  			defer cancel()
  			stream := client.Messages.NewStreaming(ctx, anthropic.MessageNewParams{
  				Model:        model,
  				MaxTokens:    64000,
  				System:       []anthropic.TextBlockParam{{Text: subagentSystem}},
  				OutputConfig: anthropic.OutputConfigParam{Effort: effort},
  				Tools:        []anthropic.ToolUnionParam{bashTool, reportTool},
  				Messages:     messages,
  			})
  			defer stream.Close()
  			for stream.Next() {
  				if err := response.Accumulate(stream.Current()); err != nil {
  					return err
  				}
  			}
  			return stream.Err()
  		}()
  		if err != nil {
  			return "", err
  		}
  		messages = append(messages, response.ToParam())
  		if response.StopReason == anthropic.StopReasonPauseTurn {
  			continue
  		}
  		if response.StopReason != anthropic.StopReasonToolUse {
  			var text strings.Builder
  			for _, block := range response.Content {
  				if textBlock, ok := block.AsAny().(anthropic.TextBlock); ok {
  					text.WriteString(textBlock.Text)
  				}
  			}
  			if response.StopReason == anthropic.StopReasonMaxTokens {
  				text.WriteString("\n\n(warning: subagent response was truncated at max_tokens)")
  			}
  			return text.String(), nil
  		}
  		var toolResults []anthropic.ContentBlockParamUnion
  		var report string
  		var reportRecorded bool
  		for _, block := range response.Content {
  			toolUse, ok := block.AsAny().(anthropic.ToolUseBlock)
  			if !ok {
  				continue
  			}
  			var output string
  			var isError bool
  			switch toolUse.Name {
  			case "report_findings":
  				report = string(toolUse.Input)
  				var pretty bytes.Buffer
  				if err := json.Indent(&pretty, toolUse.Input, "", "  "); err == nil {
  					report = pretty.String()
  				}
  				reportRecorded = true
  				output = "Findings recorded."
  			case "bash":
  				output, isError = handleBashBlock(ctx, toolUse)
  			default:
  				output, isError = fmt.Sprintf("unknown tool: %s", toolUse.Name), true
  			}
  			toolResults = append(toolResults, anthropic.NewToolResultBlock(toolUse.ID, output, isError))
  		}
  		if reportRecorded {
  			return report, nil
  		}
  		messages = append(messages, anthropic.NewUserMessage(toolResults...))
  	}
  	return "(subagent hit the turn limit before finishing)", nil
  }

  ```

  ```java Java
  // 一个子代理：一个小型嵌套代理循环，配备 bash 工具和 report_findings。
  // 子代理继承主循环的 effort 级别。
  String runSubagent(Model model, String prompt) throws InterruptedException {
      String subagentSystem = "You are one agent in a larger parallel fan-out, assigned a single subtask. "
              + "Investigate it directly, using bash to check facts rather than guessing, and finish "
              + "by calling report_findings exactly once. Return findings, not narration.";
      List<MessageParam> messages = new ArrayList<>();
      messages.add(MessageParam.builder().role(MessageParam.Role.USER).content(prompt).build());
      for (int turn = 0; turn < MAX_SUBAGENT_TURNS; turn++) {
          MessageCreateParams params = MessageCreateParams.builder()
                  .model(model)
                  .maxTokens(64000L)
                  .system(subagentSystem)
                  .outputConfig(OutputConfig.builder().effort(EFFORT).build())
                  .addTool(BASH_TOOL)
                  .addTool(REPORT_TOOL)
                  .messages(messages)
                  .build();
          MessageAccumulator accumulator = MessageAccumulator.create();
          try (var stream = client.messages().createStreaming(params, REQUEST_OPTIONS)) {
              stream.stream().forEach(accumulator::accumulate);
          }
          Message response = accumulator.message();
          messages.add(response.toParam());
          StopReason stopReason = response.stopReason().orElse(null);
          if (StopReason.PAUSE_TURN.equals(stopReason)) {
              continue;
          }
          if (!StopReason.TOOL_USE.equals(stopReason)) {
              String text = response.content().stream()
                      .flatMap(block -> block.text().stream())
                      .map(TextBlock::text)
                      .collect(Collectors.joining());
              if (StopReason.MAX_TOKENS.equals(stopReason)) {
                  text += "\n\n(warning: subagent response was truncated at max_tokens)";
              }
              return text;
          }
          List<ContentBlockParam> toolResults = new ArrayList<>();
          String report = null;
          for (ContentBlock block : response.content()) {
              if (block.toolUse().isEmpty()) {
                  continue;
              }
              ToolUseBlock toolUse = block.toolUse().get();
              ToolOutput result;
              if (toolUse.name().equals("report_findings")) {
                  report = toolUse._input().convert(JsonNode.class).toPrettyString();
                  result = new ToolOutput("Findings recorded.", false);
              } else if (toolUse.name().equals("bash")) {
                  result = handleBashBlock(toolUse);
              } else {
                  result = new ToolOutput("unknown tool: " + toolUse.name(), true);
              }
              toolResults.add(ContentBlockParam.ofToolResult(ToolResultBlockParam.builder()
                      .toolUseId(toolUse.id())
                      .content(result.output())
                      .isError(result.isError())
                      .build()));
          }
          if (report != null) {
              return report;
          }
          messages.add(MessageParam.builder()
                  .role(MessageParam.Role.USER)
                  .contentOfBlockParams(toolResults)
                  .build());
      }
      return "(subagent hit the turn limit before finishing)";
  }
  ```

  ```php PHP
  /**
   * Consume a message stream and assemble the final assistant turn from its events:
   * the full content-block list plus the stop reason, equivalent to what a
   * non-streaming create call returns.
   */
  function drainMessageStream(iterable $events): array
  {
      $stringValue = fn ($value) => $value instanceof BackedEnum ? $value->value : $value;
      $blocks = [];
      $jsonBuffers = [];
      $stopReason = null;
      foreach ($events as $event) {
          $type = $stringValue($event->type);
          if ($type === 'content_block_start') {
              $blocks[$event->index] = $event->contentBlock;
              $jsonBuffers[$event->index] = '';
          } elseif ($type === 'content_block_delta') {
              $block = $blocks[$event->index];
              $delta = $event->delta;
              $deltaType = $stringValue($delta->type);
              if ($deltaType === 'text_delta') {
                  $blocks[$event->index] = $block->withText($block->text . $delta->text);
              } elseif ($deltaType === 'input_json_delta') {
                  $jsonBuffers[$event->index] .= $delta->partialJSON;
              } elseif ($deltaType === 'thinking_delta') {
                  $blocks[$event->index] = $block->withThinking($block->thinking . $delta->thinking);
              } elseif ($deltaType === 'signature_delta') {
                  $blocks[$event->index] = $block->withSignature($delta->signature);
              }
          } elseif ($type === 'message_delta') {
              $stopReason = $stringValue($event->delta->stopReason);
          }
      }
      foreach ($jsonBuffers as $index => $buffer) {
          if ($buffer !== '' && $blocks[$index] instanceof ToolUseBlock) {
              $decoded = json_decode($buffer, true);
              $blocks[$index] = $blocks[$index]->withInput(is_array($decoded) ? $decoded : []);
          }
      }
      return [array_values($blocks), $stopReason];
  }

  /**
   * One subagent: a small nested agent loop with the bash tool plus report_findings.
   * Subagents inherit the main loop's effort level.
   */
  function runSubagent(Client $client, string $model, string $prompt): string
  {
      $subagentSystem =
          'You are one agent in a larger parallel fan-out, assigned a single subtask. '
          . 'Investigate it directly, using bash to check facts rather than guessing, and finish '
          . 'by calling report_findings exactly once. Return findings, not narration.';
      $messages = [['role' => 'user', 'content' => $prompt]];
      for ($turn = 0; $turn < MAX_SUBAGENT_TURNS; $turn++) {
          $stream = $client->messages->createStream(
              model: $model,
              maxTokens: 64000,
              system: $subagentSystem,
              outputConfig: ['effort' => EFFORT],
              tools: [BASH_TOOL, REPORT_TOOL],
              messages: $messages,
              requestOptions: ['timeout' => REQUEST_TIMEOUT_SECONDS],
          );
          [$content, $stopReason] = drainMessageStream($stream);
          $messages[] = ['role' => 'assistant', 'content' => $content];
          if ($stopReason === 'pause_turn') {
              continue;
          }
          if ($stopReason !== 'tool_use') {
              $text = '';
              foreach ($content as $block) {
                  if ($block instanceof TextBlock) {
                      $text .= $block->text;
                  }
              }
              if ($stopReason === 'max_tokens') {
                  $text .= "\n\n(warning: subagent response was truncated at max_tokens)";
              }
              return $text;
          }
          $report = null;
          $toolResults = [];
          foreach ($content as $block) {
              if (!$block instanceof ToolUseBlock) {
                  continue;
              }
              if ($block->name === 'report_findings') {
                  $report = json_encode($block->input, JSON_PRETTY_PRINT);
                  $output = 'Findings recorded.';
                  $isError = false;
              } elseif ($block->name === 'bash') {
                  [$output, $isError] = handleBashBlock($block);
              } else {
                  $output = "unknown tool: {$block->name}";
                  $isError = true;
              }
              $toolResults[] = [
                  'type' => 'tool_result',
                  'tool_use_id' => $block->id,
                  'content' => $output,
                  'is_error' => $isError,
              ];
          }
          if ($report !== null) {
              return $report;
          }
          $messages[] = ['role' => 'user', 'content' => $toolResults];
      }
      return '(subagent hit the turn limit before finishing)';
  }
  ```

  ```ruby Ruby
  # 一个子代理：一个小型嵌套代理循环，带有 bash 工具和 report_findings。
  # 子代理继承主循环的 effort 级别。
  def run_subagent(model, prompt)
    subagent_system =
      "You are one agent in a larger parallel fan-out, assigned a single subtask. " \
      "Investigate it directly, using bash to check facts rather than guessing, and finish " \
      "by calling report_findings exactly once. Return findings, not narration."
    messages = [{role: "user", content: prompt}]
    MAX_SUBAGENT_TURNS.times do
      stream = CLIENT.messages.stream(
        model: model,
        max_tokens: 64_000,
        system_: subagent_system,
        output_config: {effort: EFFORT},
        tools: [BASH_TOOL, REPORT_TOOL],
        messages: messages,
        request_options: {timeout: REQUEST_TIMEOUT_SECONDS}
      )
      response = stream.accumulated_message
      messages << {role: "assistant", content: assistant_content_param(response.content)}
      next if response.stop_reason == :pause_turn

      unless response.stop_reason == :tool_use
        text = response.content.select { |block| block.type == :text }.map(&:text).join
        text += "\n\n(warning: subagent response was truncated at max_tokens)" if response.stop_reason == :max_tokens
        return text
      end

      report = nil
      tool_results = []
      response.content.each do |block|
        next unless block.type == :tool_use

        input = parse_tool_input(block.input)
        case block.name
        when "report_findings"
          report = JSON.pretty_generate(input)
          output, is_error = "Findings recorded.", false
        when "bash"
          output, is_error = handle_bash_block(block)
        else
          output, is_error = "unknown tool: #{block.name}", true
        end
        tool_results << {
          type: "tool_result",
          tool_use_id: block.id,
          content: output,
          is_error: is_error
        }
      end
      return report unless report.nil?

      messages << {role: "user", content: tool_results}
    end
    "(subagent hit the turn limit before finishing)"
  end
  ```
</CodeGroup>

## 记录结果以便重新运行时恢复

派生出数十个子智能体的扇出从头重启的代价很高。一个小型的内容寻址日志使其具有幂等性：在分派子智能体之前，在本地 JSON 文件中查找其提示的 SHA-256，如果存在已记录的结果则直接返回。中断运行、重新运行，只有从未完成的子任务会被重新计算。该日志在多次运行之间去重，而不是在单个扇出波次内去重；删除日志文件即可重新开始。

<CodeGroup>
  ```python Python
  _journal_lock = threading.Lock()


  def _load_journal() -> dict:
      try:
          with open(JOURNAL_PATH) as file:
              return json.load(file) or {}
      except (OSError, json.JSONDecodeError):
          return {}


  def journaled(prompt: str, compute) -> str:
      """Return a cached result for this exact prompt, or compute and persist it. This
      makes the fan-out resumable: interrupt the run, rerun it, and only the subtasks
      that never finished are recomputed. Delete the journal file to start fresh."""
      key = hashlib.sha256(prompt.encode()).hexdigest()
      cached = _load_journal().get(key)
      if cached is not None:
          print(f"[journal] cache hit for {key[:12]}", file=sys.stderr)
          return cached
      result = compute()
      try:
          with _journal_lock:  # fan-out writes from many threads
              journal = _load_journal()
              journal[key] = result
              temp = f"{JOURNAL_PATH}.tmp"
              with open(temp, "w") as file:
                  json.dump(journal, file)
              os.replace(temp, JOURNAL_PATH)  # atomic on POSIX and Windows
      except OSError as error:  # the journal is best-effort; never discard a computed result
          print(f"[journal] write failed: {error}", file=sys.stderr)
      return result
  ```

  ```typescript TypeScript
  let journalWriteChain = Promise.resolve();

  async function loadJournal(): Promise<Record<string, string>> {
    try {
      return JSON.parse(await readFile(JOURNAL_PATH, "utf8")) ?? {};
    } catch (error) {
      if ((error as NodeJS.ErrnoException).code !== "ENOENT") {
        console.error(`[journal] discarding unreadable journal: ${error}`);
      }
      return {};
    }
  }

  // 为这个完全相同的提示返回缓存结果，否则计算并持久化。这
  // 使扇出可恢复：中断运行后重新运行，只有那些
  // 从未完成的子任务会被重新计算。删除日志文件即可重新开始。
  async function journaled(prompt: string, compute: () => Promise<string>): Promise<string> {
    const key = createHash("sha256").update(prompt).digest("hex");
    const cached = (await loadJournal())[key];
    if (cached !== undefined) {
      console.error(`[journal] cache hit for ${key.slice(0, 12)}`);
      return cached;
    }
    const result = await compute();
    // 将写入操作串联起来，以免并发子代理相互覆盖彼此的条目。
    // 该链始终保持 settled 状态，以免一次失败的写入破坏后续写入。
    await (journalWriteChain = journalWriteChain
      .then(async () => {
        const journal = await loadJournal();
        journal[key] = result;
        const temp = `${JOURNAL_PATH}.tmp`;
        await writeFile(temp, JSON.stringify(journal));
        await rename(temp, JOURNAL_PATH);
      })
      .catch((error) => console.error(`[journal] write failed: ${error}`)));
    return result;
  }
  ```

  ```csharp C#
  SemaphoreSlim journalLock = new(1, 1);

  async Task<Dictionary<string, string>> LoadJournal()
  {
      try
      {
          return JsonSerializer.Deserialize<Dictionary<string, string>>(await File.ReadAllTextAsync(journalPath)) ?? [];
      }
      catch (Exception error) when (error is IOException or UnauthorizedAccessException or JsonException)
      {
          return [];
      }
  }

  // 返回此确切提示的缓存结果，或计算并持久化它。这使得
  // 扇出可恢复：中断运行后重新运行，只有从未完成的子任务
  // 才会被重新计算。删除日志文件即可重新开始。
  async Task<string> Journaled(string prompt, Func<Task<string>> compute)
  {
      var key = Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(prompt))).ToLowerInvariant();
      if ((await LoadJournal()).TryGetValue(key, out var cached))
      {
          Console.Error.WriteLine($"[journal] cache hit for {key[..12]}");
          return cached;
      }
      var result = await compute();
      await journalLock.WaitAsync(); // fan-out writes from many tasks
      try
      {
          var journal = await LoadJournal();
          journal[key] = result;
          var temp = journalPath + ".tmp";
          await File.WriteAllTextAsync(temp, JsonSerializer.Serialize(journal));
          File.Move(temp, journalPath, overwrite: true);
      }
      catch (Exception error) when (error is IOException or UnauthorizedAccessException or NotSupportedException)
      {
          // 日志是尽力而为的；绝不丢弃已计算的结果。
          Console.Error.WriteLine($"[journal] write failed: {error.Message}");
      }
      finally
      {
          journalLock.Release();
      }
      return result;
  }
  ```

  ```go Go
  var journalMutex sync.Mutex

  func loadJournal() map[string]string {
  	data, err := os.ReadFile(journalPath)
  	if err != nil {
  		return map[string]string{}
  	}
  	var journal map[string]string
  	if err := json.Unmarshal(data, &journal); err != nil || journal == nil {
  		return map[string]string{}
  	}
  	return journal
  }

  // journaled 返回该确切提示的缓存结果，或计算并持久化它。
  // 这使得分发可恢复：中断运行后重新运行，只有从未完成的
  // 子任务才会被重新计算。删除日志文件即可重新开始。
  func journaled(prompt string, compute func() (string, error)) (string, error) {
  	sum := sha256.Sum256([]byte(prompt))
  	key := hex.EncodeToString(sum[:])
  	if cached, ok := loadJournal()[key]; ok {
  		fmt.Fprintf(os.Stderr, "[journal] cache hit for %s\n", key[:12])
  		return cached, nil
  	}
  	result, err := compute()
  	if err != nil {
  		return "", err
  	}
  	journalMutex.Lock() // fan-out writes from many goroutines
  	defer journalMutex.Unlock()
  	journal := loadJournal()
  	journal[key] = result
  	data, _ := json.Marshal(journal)
  	temp := journalPath + ".tmp"
  	if err := os.WriteFile(temp, data, 0o644); err != nil {
  		fmt.Fprintf(os.Stderr, "[journal] write failed: %s\n", err)
  	} else if err := os.Rename(temp, journalPath); err != nil {
  		fmt.Fprintf(os.Stderr, "[journal] write failed: %s\n", err)
  		_ = os.Remove(temp)
  	}
  	return result, nil
  }

  ```

  ```java Java
  static final ObjectMapper JOURNAL_MAPPER = new ObjectMapper();
  static final ReentrantLock JOURNAL_LOCK = new ReentrantLock();

  Map<String, String> loadJournal() {
      try {
          return Objects.requireNonNullElseGet(
                  JOURNAL_MAPPER.readValue(Files.readString(JOURNAL_PATH), new TypeReference<HashMap<String, String>>() {}),
                  HashMap::new);
      } catch (IOException error) {
          return new HashMap<>();
      }
  }

  // 返回此确切提示的缓存结果，或计算并持久化它。这使得
  // 分发过程可恢复：中断运行后重新运行，只有从未完成的子任务
  // 才会被重新计算。删除日志文件即可重新开始。
  String journaled(String prompt, Callable<String> compute) throws Exception {
      var digest = MessageDigest.getInstance("SHA-256").digest(prompt.getBytes(StandardCharsets.UTF_8));
      String key = HexFormat.of().formatHex(digest);
      String cached = loadJournal().get(key);
      if (cached != null) {
          System.err.println("[journal] cache hit for " + key.substring(0, 12));
          return cached;
      }
      String result = compute.call();
      JOURNAL_LOCK.lock(); // fan-out writes from many threads
      try {
          Map<String, String> journal = loadJournal();
          journal.put(key, result);
          Path temp = JOURNAL_PATH.resolveSibling(JOURNAL_PATH.getFileName() + ".tmp");
          Files.writeString(temp, JOURNAL_MAPPER.writeValueAsString(journal));
          Files.move(temp, JOURNAL_PATH, StandardCopyOption.REPLACE_EXISTING, StandardCopyOption.ATOMIC_MOVE);
      } catch (IOException error) {
          // 日志记录是尽力而为的；切勿丢弃已计算的结果。
          System.err.println("[journal] write failed: " + error);
      } finally {
          JOURNAL_LOCK.unlock();
      }
      return result;
  }
  ```

  ```php PHP
  function loadJournal(): array
  {
      $raw = @file_get_contents(JOURNAL_PATH);
      if ($raw === false) {
          return [];
      }
      $decoded = json_decode($raw, true);
      return is_array($decoded) ? $decoded : [];
  }

  /**
   * Return a cached result for this exact prompt, or compute and persist it. This
   * makes the fan-out resumable: interrupt the run, rerun it, and only the subtasks
   * that never finished are recomputed. Delete the journal file to start fresh.
   */
  function journaled(string $prompt, callable $compute): string
  {
      $key = hash('sha256', $prompt);
      $journal = loadJournal();
      if (array_key_exists($key, $journal)) {
          fwrite(STDERR, '[journal] cache hit for ' . substr($key, 0, 12) . "\n");
          return $journal[$key];
      }
      $result = $compute();
      $journal = loadJournal();
      $journal[$key] = $result;
      $temp = JOURNAL_PATH . '.tmp';
      $encoded = json_encode($journal, JSON_INVALID_UTF8_SUBSTITUTE);
      if ($encoded === false || @file_put_contents($temp, $encoded) === false || !@rename($temp, JOURNAL_PATH)) {
          fwrite(STDERR, '[journal] write failed: ' . (error_get_last()['message'] ?? json_last_error_msg()) . "\n");
          @unlink($temp);
      }
      return $result;
  }
  ```

  ```ruby Ruby
  JOURNAL_LOCK = Mutex.new

  def load_journal
    JSON.parse(File.read(JOURNAL_PATH)) || {}
  rescue SystemCallError, JSON::ParserError
    {}
  end

  # 返回此提示的缓存结果，若无则计算并持久化。这
  # 使得分发（fan-out）可以恢复：中断运行后重新运行，只有那些
  # 从未完成的子任务会被重新计算。删除日志（journal）文件即可重新开始。
  def journaled(prompt)
    key = Digest::SHA256.hexdigest(prompt)
    cached = load_journal[key]
    unless cached.nil?
      warn "[journal] cache hit for #{key[0, 12]}"
      return cached
    end
    result = yield
    begin
      JOURNAL_LOCK.synchronize do # fan-out writes from many threads
        journal = load_journal
        journal[key] = result
        temp = "#{JOURNAL_PATH}.tmp"
        File.write(temp, JSON.generate(journal))
        File.rename(temp, JOURNAL_PATH)
      end
    rescue SystemCallError => error # the journal is best-effort; never discard a computed result
      warn "[journal] write failed: #{error}"
    end
    result
  end
  ```
</CodeGroup>

## 扇出，然后验证

扇出最多接受 `MAX_TOTAL_SUBTASKS` 个提示，通过日志运行它们，同时最多有 `MAX_CONCURRENT` 个在执行中（PHP 移植版为顺序执行），并隔离故障，使一个出错的子智能体降级为一个错误字符串，而不是终止整个运行。第一波完成后，第二波复用相同的子智能体路径来尝试反驳每个结果：每个验证者都从源头重新推导各项论断，在不确定时默认判定为已反驳。原始结果及其裁定都会返回给编排器，以便它能够综合权衡。

<CodeGroup>
  ```python Python
  def normalize_subtasks(raw) -> list[str]:
      """Accept the subtasks input in whatever shape the model emits: an array, the array
      JSON-encoded as a single string, or a newline-separated list."""
      if isinstance(raw, str):
          try:
              raw = json.loads(raw)
          except json.JSONDecodeError:
              raw = raw.splitlines() if "\n" in raw else [raw]
      if not isinstance(raw, list):
          return []
      return [task.strip() for task in raw if isinstance(task, str) and task.strip()]


  def verify_prompt_for(subtask: str, result: str) -> str:
      return (
          "Adversarially verify the subagent result below: try to REFUTE it. Re-derive the "
          "claims yourself with bash rather than trusting the result, and look for evidence "
          "that contradicts them. Default to refuted if uncertain. Call report_findings with "
          "summary 'refuted: <why>' or 'confirmed: <why>', citing the file:line or command "
          "output that decided it.\n\n"
          f"Subtask: {subtask}\n\nResult to verify:\n{result}"
      )


  def run_workflow(model: str, raw_subtasks) -> tuple[str, bool]:
      """Run subtasks as parallel subagents, then run a second verification wave over
      the results, and return both. MAX_TOTAL_SUBTASKS bounds how many the model can
      queue; MAX_CONCURRENT bounds how many run at once."""
      all_subtasks = normalize_subtasks(raw_subtasks)
      subtasks = all_subtasks[:MAX_TOTAL_SUBTASKS]
      dropped = len(all_subtasks) - len(subtasks)
      if not subtasks:
          return "Workflow error: no usable subtasks were provided.", True
      print(f"[workflow] fanning out {len(subtasks)} agents", file=sys.stderr)

      def run_one(prompt: str) -> str:
          try:
              return journaled(prompt, lambda: run_subagent(model, prompt))
          except Exception as error:  # isolation boundary: one bad subagent should not end the run
              return f"(subagent failed: {type(error).__name__}: {error})"

      with concurrent.futures.ThreadPoolExecutor(max_workers=MAX_CONCURRENT) as pool:
          results = list(pool.map(run_one, subtasks))
          print(f"[workflow] verifying {len(results)} results", file=sys.stderr)
          verify_prompts = [verify_prompt_for(task, result) for task, result in zip(subtasks, results)]
          verdicts = list(pool.map(run_one, verify_prompts))

      joined = "\n\n".join(
          f"[agent {index + 1}: {task}]\n{result}\n\n[verify {index + 1}]\n{verdict}"
          for index, (task, result, verdict) in enumerate(zip(subtasks, results, verdicts))
      )
      if dropped > 0:
          joined = (
              f"(note: {dropped} subtasks beyond MAX_TOTAL_SUBTASKS={MAX_TOTAL_SUBTASKS} were not "
              "run; rerun them in a follow-up Workflow call)\n\n" + joined
          )
      return joined, False
  ```

  ```typescript TypeScript
  // 接受模型以任何形式输出的 subtasks 输入：数组、
  // 被 JSON 编码为单个字符串的数组，或以换行符分隔的列表。
  function normalizeSubtasks(raw: unknown): string[] {
    let value = raw;
    if (typeof raw === "string") {
      try {
        value = JSON.parse(raw);
      } catch {
        value = raw.includes("\n") ? raw.split("\n") : [raw];
      }
    }
    if (!Array.isArray(value)) {
      return [];
    }
    return value
      .filter((task): task is string => typeof task === "string")
      .map((task) => task.trim())
      .filter((task) => task.length > 0);
  }

  function verifyPromptFor(subtask: string, result: string): string {
    return (
      "Adversarially verify the subagent result below: try to REFUTE it. Re-derive the " +
      "claims yourself with bash rather than trusting the result, and look for evidence " +
      "that contradicts them. Default to refuted if uncertain. Call report_findings with " +
      "summary 'refuted: <why>' or 'confirmed: <why>', citing the file:line or command " +
      "output that decided it.\n\n" +
      `Subtask: ${subtask}\n\nResult to verify:\n${result}`
    );
  }

  // 带并发限制的 map：同一时刻最多有 `limit` 个任务在执行。
  async function mapWithLimit<In, Out>(
    items: readonly In[],
    limit: number,
    task: (item: In) => Promise<Out>,
  ): Promise<Out[]> {
    const results = new Array<Out>(items.length);
    let cursor = 0;
    const workers = Array.from({ length: Math.min(limit, items.length) }, async () => {
      while (cursor < items.length) {
        const index = cursor++;
        results[index] = await task(items[index]);
      }
    });
    await Promise.all(workers);
    return results;
  }

  // 将子任务作为并行子代理运行，然后对结果运行第二轮
  // 验证，并返回两者。MAX_TOTAL_SUBTASKS 限制模型可以
  // 排队的数量；MAX_CONCURRENT 限制同时运行的数量。
  async function runWorkflow(
    model: string,
    rawSubtasks: unknown,
  ): Promise<{ output: string; isError: boolean }> {
    const allSubtasks = normalizeSubtasks(rawSubtasks);
    const subtasks = allSubtasks.slice(0, MAX_TOTAL_SUBTASKS);
    const dropped = allSubtasks.length - subtasks.length;
    if (subtasks.length === 0) {
      return { output: "Workflow error: no usable subtasks were provided.", isError: true };
    }
    console.error(`[workflow] fanning out ${subtasks.length} agents`);

    const runOne = async (prompt: string): Promise<string> => {
      try {
        return await journaled(prompt, () => runSubagent(model, prompt));
      } catch (error) {
        // 隔离边界：一个出错的子代理不应终止整个运行。
        const reason = error instanceof Error ? `${error.name}: ${error.message}` : String(error);
        return `(subagent failed: ${reason})`;
      }
    };

    const results = await mapWithLimit(subtasks, MAX_CONCURRENT, runOne);
    console.error(`[workflow] verifying ${results.length} results`);
    const verifyPrompts = subtasks.map((task, index) => verifyPromptFor(task, results[index]));
    const verdicts = await mapWithLimit(verifyPrompts, MAX_CONCURRENT, runOne);

    let joined = subtasks
      .map(
        (task, index) =>
          `[agent ${index + 1}: ${task}]\n${results[index]}\n\n[verify ${index + 1}]\n${verdicts[index]}`,
      )
      .join("\n\n");
    if (dropped > 0) {
      joined =
        `(note: ${dropped} subtasks beyond MAX_TOTAL_SUBTASKS=${MAX_TOTAL_SUBTASKS} were not ` +
        "run; rerun them in a follow-up Workflow call)\n\n" +
        joined;
    }
    return { output: joined, isError: false };
  }
  ```

  ```csharp C#
  // 接受模型以任何形式输出的 subtasks 输入：数组、作为单个字符串
  // JSON 编码的数组，或以换行符分隔的列表。
  List<string> NormalizeSubtasks(JsonElement raw)
  {
      List<string> tasks = [];
      if (raw.ValueKind == JsonValueKind.Array)
      {
          tasks = raw.EnumerateArray()
              .Where(item => item.ValueKind == JsonValueKind.String)
              .Select(item => item.GetString()!)
              .ToList();
      }
      else if (raw.ValueKind == JsonValueKind.String)
      {
          var single = raw.GetString()!;
          try
          {
              tasks = JsonSerializer.Deserialize<List<string>>(single) ?? [];
          }
          catch (JsonException)
          {
              tasks = [.. single.Split('\n')];
          }
      }
      return tasks.Where(task => task != null).Select(task => task.Trim()).Where(task => task.Length > 0).ToList();
  }

  string VerifyPromptFor(string subtask, string result) =>
      "Adversarially verify the subagent result below: try to REFUTE it. Re-derive the "
      + "claims yourself with bash rather than trusting the result, and look for evidence "
      + "that contradicts them. Default to refuted if uncertain. Call report_findings with "
      + "summary 'refuted: <why>' or 'confirmed: <why>', citing the file:line or command "
      + "output that decided it.\n\n"
      + $"Subtask: {subtask}\n\nResult to verify:\n{result}";

  // 将子任务作为并行子代理运行，然后对结果运行第二轮验证，
  // 并返回两者。maxTotalSubtasks 限制模型可排队的数量；
  // maxConcurrent 限制同时运行的数量。
  async Task<(string Output, bool IsError)> RunWorkflow(JsonElement rawSubtasks)
  {
      var allSubtasks = NormalizeSubtasks(rawSubtasks);
      var subtasks = allSubtasks.Take(maxTotalSubtasks).ToList();
      var dropped = allSubtasks.Count - subtasks.Count;
      if (subtasks.Count == 0)
      {
          return ("Workflow error: no usable subtasks were provided.", true);
      }
      Console.Error.WriteLine($"[workflow] fanning out {subtasks.Count} agents");

      using SemaphoreSlim gate = new(maxConcurrent);
      async Task<string> RunOne(string prompt)
      {
          await gate.WaitAsync();
          try
          {
              return await Journaled(prompt, () => RunSubagent(prompt));
          }
          catch (Exception error)
          {
              // 隔离边界：一个出错的子代理不应终止整个运行。
              return $"(subagent failed: {error.GetType().Name}: {error.Message})";
          }
          finally
          {
              gate.Release();
          }
      }

      var results = await Task.WhenAll(subtasks.Select(RunOne));
      Console.Error.WriteLine($"[workflow] verifying {results.Length} results");
      var verifyPrompts = subtasks.Select((task, index) => VerifyPromptFor(task, results[index])).ToList();
      var verdicts = await Task.WhenAll(verifyPrompts.Select(RunOne));

      var joined = string.Join(
          "\n\n",
          subtasks.Select((task, index) =>
              $"[agent {index + 1}: {task}]\n{results[index]}\n\n[verify {index + 1}]\n{verdicts[index]}"));
      if (dropped > 0)
      {
          joined = $"(note: {dropped} subtasks beyond maxTotalSubtasks={maxTotalSubtasks} were not run; "
              + "rerun them in a follow-up Workflow call)\n\n" + joined;
      }
      return (joined, false);
  }
  ```

  ```go Go
  // normalizeSubtasks 接受模型以任意形式输出的 subtasks 输入：
  // 数组、被 JSON 编码为单个字符串的数组，或换行分隔的列表。
  func normalizeSubtasks(raw json.RawMessage) []string {
  	var tasks []string
  	if err := json.Unmarshal(raw, &tasks); err != nil {
  		var single string
  		if err := json.Unmarshal(raw, &single); err != nil {
  			return nil
  		}
  		if err := json.Unmarshal([]byte(single), &tasks); err != nil {
  			tasks = strings.Split(single, "\n")
  		}
  	}
  	cleaned := make([]string, 0, len(tasks))
  	for _, task := range tasks {
  		if trimmed := strings.TrimSpace(task); trimmed != "" {
  			cleaned = append(cleaned, trimmed)
  		}
  	}
  	return cleaned
  }

  func verifyPromptFor(subtask, result string) string {
  	return "Adversarially verify the subagent result below: try to REFUTE it. Re-derive the " +
  		"claims yourself with bash rather than trusting the result, and look for evidence " +
  		"that contradicts them. Default to refuted if uncertain. Call report_findings with " +
  		"summary 'refuted: <why>' or 'confirmed: <why>', citing the file:line or command " +
  		"output that decided it.\n\n" +
  		"Subtask: " + subtask + "\n\nResult to verify:\n" + result
  }

  // mapWithLimit 对 items 运行 task，同时运行的 goroutine 不超过 limit 个。
  func mapWithLimit(items []string, limit int, task func(string) string) []string {
  	results := make([]string, len(items))
  	semaphore := make(chan struct{}, limit)
  	var waitGroup sync.WaitGroup
  	for index, item := range items {
  		waitGroup.Add(1)
  		semaphore <- struct{}{}
  		go func() {
  			defer waitGroup.Done()
  			defer func() { <-semaphore }()
  			results[index] = task(item)
  		}()
  	}
  	waitGroup.Wait()
  	return results
  }

  // runWorkflow 将子任务作为并行子代理运行，然后对结果运行第二轮验证，
  // 并返回两者。maxTotalSubtasks 限制模型可排队的数量；
  // maxConcurrent 限制同时运行的数量。
  func runWorkflow(ctx context.Context, model string, rawSubtasks json.RawMessage) (string, bool) {
  	allSubtasks := normalizeSubtasks(rawSubtasks)
  	subtasks := allSubtasks
  	if len(subtasks) > maxTotalSubtasks {
  		subtasks = subtasks[:maxTotalSubtasks]
  	}
  	dropped := len(allSubtasks) - len(subtasks)
  	if len(subtasks) == 0 {
  		return "Workflow error: no usable subtasks were provided.", true
  	}
  	fmt.Fprintf(os.Stderr, "[workflow] fanning out %d agents\n", len(subtasks))

  	runOne := func(prompt string) string {
  		report, err := journaled(prompt, func() (string, error) { return runSubagent(ctx, model, prompt) })
  		if err != nil {
  			// 隔离边界：一个出错的子代理不应终止整个运行。
  			return fmt.Sprintf("(subagent failed: %s)", err)
  		}
  		return report
  	}

  	results := mapWithLimit(subtasks, maxConcurrent, runOne)
  	fmt.Fprintf(os.Stderr, "[workflow] verifying %d results\n", len(results))
  	verifyPrompts := make([]string, len(subtasks))
  	for index, task := range subtasks {
  		verifyPrompts[index] = verifyPromptFor(task, results[index])
  	}
  	verdicts := mapWithLimit(verifyPrompts, maxConcurrent, runOne)

  	sections := make([]string, len(subtasks))
  	for index, task := range subtasks {
  		sections[index] = fmt.Sprintf("[agent %d: %s]\n%s\n\n[verify %d]\n%s",
  			index+1, task, results[index], index+1, verdicts[index])
  	}
  	joined := strings.Join(sections, "\n\n")
  	if dropped > 0 {
  		joined = fmt.Sprintf("(note: %d subtasks beyond maxTotalSubtasks=%d were not run; "+
  			"rerun them in a follow-up Workflow call)\n\n", dropped, maxTotalSubtasks) + joined
  	}
  	return joined, false
  }

  ```

  ```java Java
  // 接受模型输出的任何形式的 subtasks 输入：数组、以单个字符串
  // JSON 编码的数组，或以换行符分隔的列表。
  List<String> normalizeSubtasks(JsonValue raw) {
      List<String> tasks = new ArrayList<>();
      if (raw.asArray().isPresent()) {
          for (JsonValue item : (List<JsonValue>) raw.asArray().get()) {
              tasks.add(item.asString().isPresent() ? item.asStringOrThrow() : item.toString());
          }
      } else if (raw.asString().isPresent()) {
          String single = raw.asStringOrThrow();
          try {
              String[] parsed = new ObjectMapper().readValue(single, String[].class);
              if (parsed != null) {
                  for (String task : parsed) {
                      tasks.add(task);
                  }
              }
          } catch (JsonProcessingException error) {
              for (String task : single.split("\n")) {
                  tasks.add(task);
              }
          }
      }
      return tasks.stream()
              .filter(task -> task != null)
              .map(String::trim)
              .filter(task -> !task.isEmpty())
              .toList();
  }

  String verifyPromptFor(String subtask, String result) {
      return "Adversarially verify the subagent result below: try to REFUTE it. Re-derive the "
              + "claims yourself with bash rather than trusting the result, and look for evidence "
              + "that contradicts them. Default to refuted if uncertain. Call report_findings with "
              + "summary 'refuted: <why>' or 'confirmed: <why>', citing the file:line or command "
              + "output that decided it.\n\n"
              + "Subtask: " + subtask + "\n\nResult to verify:\n" + result;
  }

  List<String> runAll(ExecutorService pool, List<String> prompts, Model model) throws InterruptedException {
      List<Callable<String>> jobs = prompts.stream()
              .<Callable<String>>map(prompt -> () -> journaled(prompt, () -> runSubagent(model, prompt)))
              .toList();
      List<String> results = new ArrayList<>();
      for (Future<String> future : pool.invokeAll(jobs)) {
          try {
              results.add(future.get());
          } catch (ExecutionException | CancellationException error) {
              // 隔离边界：一个出错的子代理不应终止整个运行。
              Throwable cause = error.getCause() != null ? error.getCause() : error;
              results.add("(subagent failed: " + cause + ")");
          }
      }
      return results;
  }

  // 将子任务作为并行子代理运行，然后对结果执行第二轮验证，
  // 并返回两者。MAX_TOTAL_SUBTASKS 限制模型可排队的数量；
  // MAX_CONCURRENT 限制同时运行的数量。
  ToolOutput runWorkflow(Model model, JsonValue rawSubtasks) throws InterruptedException {
      List<String> allSubtasks = normalizeSubtasks(rawSubtasks);
      List<String> subtasks = allSubtasks.stream().limit(MAX_TOTAL_SUBTASKS).toList();
      int dropped = allSubtasks.size() - subtasks.size();
      if (subtasks.isEmpty()) {
          return new ToolOutput("Workflow error: no usable subtasks were provided.", true);
      }
      System.err.println("[workflow] fanning out " + subtasks.size() + " agents");

      List<String> results;
      List<String> verdicts;
      try (ExecutorService pool = Executors.newFixedThreadPool(MAX_CONCURRENT, Thread.ofVirtual().factory())) {
          results = runAll(pool, subtasks, model);
          System.err.println("[workflow] verifying " + results.size() + " results");
          List<String> verifyPrompts = IntStream.range(0, subtasks.size())
                  .mapToObj(index -> verifyPromptFor(subtasks.get(index), results.get(index)))
                  .toList();
          verdicts = runAll(pool, verifyPrompts, model);
      }
      String joined = IntStream.range(0, subtasks.size())
              .mapToObj(index -> "[agent " + (index + 1) + ": " + subtasks.get(index) + "]\n" + results.get(index)
                      + "\n\n[verify " + (index + 1) + "]\n" + verdicts.get(index))
              .collect(Collectors.joining("\n\n"));
      if (dropped > 0) {
          joined = "(note: " + dropped + " subtasks beyond MAX_TOTAL_SUBTASKS=" + MAX_TOTAL_SUBTASKS
                  + " were not run; rerun them in a follow-up Workflow call)\n\n" + joined;
      }
      return new ToolOutput(joined, false);
  }
  ```

  ```php PHP
  /**
   * Accept the subtasks input in whatever shape the model emits: an array, the array
   * JSON-encoded as a single string, or a newline-separated list.
   */
  function normalizeSubtasks(mixed $raw): array
  {
      if (is_string($raw)) {
          try {
              $raw = json_decode($raw, true, flags: JSON_THROW_ON_ERROR);
          } catch (JsonException) {
              $raw = str_contains($raw, "\n") ? explode("\n", $raw) : [$raw];
          }
      }
      if (!is_array($raw)) {
          return [];
      }
      $tasks = array_map('trim', array_filter($raw, 'is_string'));
      return array_values(array_filter($tasks, fn ($task) => $task !== ''));
  }

  function verifyPromptFor(string $subtask, string $result): string
  {
      return 'Adversarially verify the subagent result below: try to REFUTE it. Re-derive the '
          . 'claims yourself with bash rather than trusting the result, and look for evidence '
          . 'that contradicts them. Default to refuted if uncertain. Call report_findings with '
          . "summary 'refuted: <why>' or 'confirmed: <why>', citing the file:line or command "
          . "output that decided it.\n\n"
          . "Subtask: {$subtask}\n\nResult to verify:\n{$result}";
  }

  /**
   * Run subtasks through the journal, then run a second verification wave over the
   * results, and return both. PHP's standard runtime has no lightweight thread pool,
   * so both waves run sequentially here (MAX_CONCURRENT is unused); the SDK examples
   * in other languages fan them out in parallel.
   */
  function runWorkflow(Client $client, string $model, mixed $rawSubtasks): array
  {
      $allSubtasks = normalizeSubtasks($rawSubtasks);
      $subtasks = array_slice($allSubtasks, 0, MAX_TOTAL_SUBTASKS);
      $dropped = count($allSubtasks) - count($subtasks);
      if ($subtasks === []) {
          return ['Workflow error: no usable subtasks were provided.', true];
      }
      fwrite(STDERR, '[workflow] running ' . count($subtasks) . " agents\n");

      $runOne = function (string $prompt) use ($client, $model): string {
          try {
              return journaled($prompt, fn () => runSubagent($client, $model, $prompt));
          } catch (Throwable $error) {
              // 隔离边界：一个出错的子代理不应终止整个运行。
              return '(subagent failed: ' . $error::class . ': ' . $error->getMessage() . ')';
          }
      };

      $results = array_map($runOne, $subtasks);
      fwrite(STDERR, '[workflow] verifying ' . count($results) . " results\n");
      $verifyPrompts = array_map(verifyPromptFor(...), $subtasks, $results);
      $verdicts = array_map($runOne, $verifyPrompts);

      $sections = [];
      foreach ($subtasks as $index => $task) {
          $sections[] = '[agent ' . ($index + 1) . ": {$task}]\n{$results[$index]}"
              . "\n\n[verify " . ($index + 1) . "]\n{$verdicts[$index]}";
      }
      $joined = implode("\n\n", $sections);
      if ($dropped > 0) {
          $joined = '(note: ' . $dropped . ' subtasks beyond MAX_TOTAL_SUBTASKS=' . MAX_TOTAL_SUBTASKS
              . " were not run; rerun them in a follow-up Workflow call)\n\n" . $joined;
      }
      return [$joined, false];
  }
  ```

  ```ruby Ruby
  # 接受模型以任何形式输出的 subtasks 输入：数组、被 JSON 编码为
  # 单个字符串的数组，或以换行符分隔的列表。
  def normalize_subtasks(raw)
    if raw.is_a?(String)
      begin
        raw = JSON.parse(raw)
      rescue JSON::ParserError
        raw = raw.include?("\n") ? raw.split("\n") : [raw]
      end
    end
    return [] unless raw.is_a?(Array)
    raw.select { |task| task.is_a?(String) }.map(&:strip).reject(&:empty?)
  end

  def verify_prompt_for(subtask, result)
    "Adversarially verify the subagent result below: try to REFUTE it. Re-derive the " \
      "claims yourself with bash rather than trusting the result, and look for evidence " \
      "that contradicts them. Default to refuted if uncertain. Call report_findings with " \
      "summary 'refuted: <why>' or 'confirmed: <why>', citing the file:line or command " \
      "output that decided it.\n\n" \
      "Subtask: #{subtask}\n\nResult to verify:\n#{result}"
  end

  # 带并发限制的映射：同一时刻最多有 `limit` 个线程在运行。
  def map_with_limit(items, limit)
    results = Array.new(items.length)
    queue = Queue.new
    items.each_with_index { |item, index| queue << [index, item] }
    workers = Array.new([limit, items.length].min) do
      Thread.new do
        until queue.empty?
          index, item = queue.pop(true) rescue break
          results[index] = yield item
        end
      end
    end
    workers.each(&:join)
    results
  end

  # 将子任务作为并行子代理运行，然后对结果运行第二轮验证，
  # 并返回两者。MAX_TOTAL_SUBTASKS 限制模型可以排队的数量；
  # MAX_CONCURRENT 限制同时运行的数量。
  def run_workflow(model, raw_subtasks)
    all_subtasks = normalize_subtasks(raw_subtasks)
    subtasks = all_subtasks.first(MAX_TOTAL_SUBTASKS)
    dropped = all_subtasks.length - subtasks.length
    return ["Workflow error: no usable subtasks were provided.", true] if subtasks.empty?

    warn "[workflow] fanning out #{subtasks.length} agents"
    run_one = lambda do |prompt|
      journaled(prompt) { run_subagent(model, prompt) }
    rescue => error # isolation boundary: one bad subagent should not end the run
      "(subagent failed: #{error.class}: #{error.message})"
    end

    results = map_with_limit(subtasks, MAX_CONCURRENT, &run_one)
    warn "[workflow] verifying #{results.length} results"
    verify_prompts = subtasks.zip(results).map { |task, result| verify_prompt_for(task, result) }
    verdicts = map_with_limit(verify_prompts, MAX_CONCURRENT, &run_one)

    joined = subtasks.each_with_index.map do |task, index|
      "[agent #{index + 1}: #{task}]\n#{results[index]}\n\n[verify #{index + 1}]\n#{verdicts[index]}"
    end.join("\n\n")
    if dropped > 0
      joined =
        "(note: #{dropped} subtasks beyond MAX_TOTAL_SUBTASKS=#{MAX_TOTAL_SUBTASKS} were not " \
        "run; rerun them in a follow-up Workflow call)\n\n#{joined}"
    end
    [joined, false]
  end
  ```
</CodeGroup>

## 使用对话中途系统消息切换模式

智能体首先追加用户的消息，然后追加任何应发送的系统消息：退出通知、进入时的完整模式文本，或定期的复习提醒。将系统消息放在用户轮次之后，可以使其前面的每个缓存字节保持不变，并满足系统消息须跟在用户轮次之后的放置规则。

<CodeGroup>
  ```bash cURL
  # 单个编排模式轮次：模式提醒在 messages 数组中作为一个
  # {"role": "system"} 条目，放在其所适用的用户轮次之后。响应
  # 在第一个工具调用处停止。执行工具调用并分发
  # 子代理的代理循环展示在 SDK 标签页中；此处的 Workflow 描述经过精简，
  # SDK 示例中包含完整的长期同意（standing-consent）文本。
  curl --fail-with-body -sS https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d @- <<'EOF'
  {
    "model": "claude-opus-5",
    "max_tokens": 64000,
    "system": "You are a helpful general-purpose agent. Answer the user's request directly.",
    "output_config": {"effort": "xhigh"},
    "tools": [
      {
        "name": "Workflow",
        "description": "Orchestrate a multiagent workflow: split a large task into independent subtasks and run them as parallel agents, then collect their results. Opt-in: only use this tool when the user explicitly asks for a workflow, or when a system message confirms that orchestration mode is on. Granularity: scope each subtask to a distinct concern rather than per line or file section; a review of a small module rarely needs more than about ten. Standing consent: while a system message confirms orchestration mode is on, author and run a workflow for every substantive task by default; work solo only on conversational turns or trivial mechanical edits.",
        "input_schema": {
          "type": "object",
          "properties": {
            "subtasks": {
              "type": "array",
              "items": {"type": "string"},
              "description": "Independent subtask prompts to run as parallel agents"
            }
          },
          "required": ["subtasks"]
        }
      },
      {"type": "bash_20250124", "name": "bash"}
    ],
    "messages": [
      {
        "role": "user",
        "content": "Explore the current directory, then give a thorough review: what it does, code-quality issues, and concrete improvements."
      },
      {
        "role": "system",
        "content": "Orchestration mode is on: optimize for the most exhaustive, correct answer rather than the fastest one. Use the Workflow tool on every substantive task, sized to the problem's natural decomposition rather than the maximum the tool allows. See the Workflow tool's description for standing consent, granularity guidance, and quality patterns. Work solo only on conversational or trivial turns."
      }
    ]
  }
  EOF
  ```

  ```bash CLI
  # 一个编排模式轮次：模式提醒作为 system 角色条目放在 messages 数组中，
  # 位于其所适用的用户轮次之后。响应在第一个工具调用处停止。
  # 执行工具调用并分发子代理的代理循环
  # 在 SDK 标签页中展示；此处的 Workflow 描述经过精简，
  # 完整的长期同意文本见 SDK 示例。
  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 64000
  system: You are a helpful general-purpose agent. Answer the user's request directly.
  output_config: {effort: xhigh}
  tools:
    - name: Workflow
      description: >-
        Orchestrate a multiagent workflow: split a large task into independent
        subtasks and run them as parallel agents, then collect their results.
        Opt-in: only use this tool when the user explicitly asks for a workflow,
        or when a system message confirms that orchestration mode is on.
        Granularity: scope each subtask to a distinct concern rather than per
        line or file section; a review of a small module rarely needs more than
        about ten. Standing consent: while a system message confirms
        orchestration mode is on, author and run a workflow for every
        substantive task by default; work solo only on conversational turns or
        trivial mechanical edits.
      input_schema:
        type: object
        properties:
          subtasks:
            type: array
            items: {type: string}
            description: Independent subtask prompts to run as parallel agents
        required: [subtasks]
    - {type: bash_20250124, name: bash}
  messages:
    - role: user
      content: >-
        Explore the current directory, then give a thorough review: what it
        does, code-quality issues, and concrete improvements.
    - role: system
      content: >-
        Orchestration mode is on: optimize for the most exhaustive, correct
        answer rather than the fastest one. Use the Workflow tool on every
        substantive task, sized to the problem's natural decomposition rather
        than the maximum the tool allows. See the Workflow tool's description
        for standing consent, granularity guidance, and quality patterns. Work
        solo only on conversational or trivial turns.
  YAML
  ```

  ```python Python
  class ModeAgent:
      """An agent loop whose orchestration mode is toggled with mid-conversation system messages."""

      def __init__(self, model: str, mode_on: bool = True):
          self.model = model
          self.mode_on = mode_on
          self.messages: list[dict] = []
          self._mode_announced = False
          self._exit_pending = False
          self._turns_since_reminder = 0

      def set_mode(self, mode_on: bool) -> None:
          """Turn the mode on or off. The notice is delivered with the next user turn."""
          if mode_on == self.mode_on:
              return
          if not mode_on:
              if self._mode_announced:
                  self._exit_pending = True
          else:
              self._exit_pending = False
          self.mode_on = mode_on

      def _due_system_messages(self) -> list[dict]:
          """System messages owed on this turn: an exit notice, the full mode text on entry,
          or a one-line refresher every TURNS_BETWEEN_REFRESHERS user turns."""
          due = []
          if self._exit_pending:
              self._exit_pending = False
              self._mode_announced = False
              due.append({"role": "system", "content": MODE_EXIT})
          if self.mode_on:
              if not self._mode_announced:
                  self._mode_announced = True
                  self._turns_since_reminder = 0
                  due.append({"role": "system", "content": MODE_ENTER})
              elif self._turns_since_reminder >= TURNS_BETWEEN_REFRESHERS:
                  self._turns_since_reminder = 0
                  due.append({"role": "system", "content": MODE_REFRESH})
          return due

      def turn(self, user_input: str) -> str:
          # 对话中途的系统消息紧跟在其所适用的用户轮次之后，这样可以
          # 保持其前面已缓存的前缀不受影响。
          self.messages.append({"role": "user", "content": user_input})
          self.messages.extend(self._due_system_messages())
          self._turns_since_reminder += 1

          for _ in range(MAX_MAIN_TURNS):
              with client.messages.stream(
                  model=self.model,
                  max_tokens=64000,
                  system=SYSTEM_PROMPT,  # static for the whole session
                  output_config={"effort": EFFORT},
                  tools=[WORKFLOW_TOOL, BASH_TOOL],
                  messages=self.messages,
                  timeout=REQUEST_TIMEOUT_SECONDS,
              ) as stream:
                  response = stream.get_final_message()
              self.messages.append({"role": "assistant", "content": response.content})

              if response.stop_reason == "pause_turn":
                  continue
              if response.stop_reason != "tool_use":
                  text = "".join(block.text for block in response.content if block.type == "text")
                  if response.stop_reason == "max_tokens":
                      # 丢弃被截断的助手消息，以免后续轮次在其基础上继续构建。
                      self.messages.pop()
                      text += "\n\n(warning: response was truncated at max_tokens)"
                  return text

              tool_results = []
              for block in response.content:
                  if block.type != "tool_use":
                      continue
                  if block.name == "Workflow":
                      output, is_error = run_workflow(self.model, block.input.get("subtasks", []))
                  elif block.name == "bash":
                      output, is_error = handle_bash_block(block)
                  else:
                      output, is_error = f"unknown tool: {block.name}", True
                  tool_results.append(
                      {
                          "type": "tool_result",
                          "tool_use_id": block.id,
                          "content": output,
                          "is_error": is_error,
                      }
                  )
              self.messages.append({"role": "user", "content": tool_results})
          return "(hit the main loop turn limit before finishing)"
  ```

  ```typescript TypeScript
  // 一个代理循环，其编排模式通过对话中途的系统消息来切换。
  class ModeAgent {
    private readonly model: string;
    private modeOn: boolean;
    private readonly messages: Anthropic.MessageParam[] = [];
    private modeAnnounced = false;
    private exitPending = false;
    private turnsSinceReminder = 0;

    constructor(model: string, modeOn = true) {
      this.model = model;
      this.modeOn = modeOn;
    }

    // 开启或关闭该模式。通知会随下一个用户轮次一起发送。
    setMode(modeOn: boolean): void {
      if (modeOn === this.modeOn) {
        return;
      }
      if (!modeOn) {
        if (this.modeAnnounced) {
          this.exitPending = true;
        }
      } else {
        this.exitPending = false;
      }
      this.modeOn = modeOn;
    }

    // 本轮应发送的系统消息：退出通知、进入时的完整模式文本，
    // 或每隔 TURNS_BETWEEN_REFRESHERS 个用户轮次发送一行提醒。
    private dueSystemMessages(): Anthropic.MessageParam[] {
      const due: Array<{ role: "system"; content: string }> = [];
      if (this.exitPending) {
        this.exitPending = false;
        this.modeAnnounced = false;
        due.push({ role: "system", content: MODE_EXIT });
      }
      if (this.modeOn) {
        if (!this.modeAnnounced) {
          this.modeAnnounced = true;
          this.turnsSinceReminder = 0;
          due.push({ role: "system", content: MODE_ENTER });
        } else if (this.turnsSinceReminder >= TURNS_BETWEEN_REFRESHERS) {
          this.turnsSinceReminder = 0;
          due.push({ role: "system", content: MODE_REFRESH });
        }
      }
      // 已发布的 SDK 将消息角色类型定义为 "user" | "assistant"；对
      // 对话中途系统消息的类型支持将随包含该功能的 SDK 版本一起发布。
      return due as unknown as Anthropic.MessageParam[];
    }

    async turn(userInput: string): Promise<string> {
      // 对话中途的系统消息紧跟在其所适用的用户轮次之后，这样
      // 可以保持其前面已缓存的前缀不受影响。
      this.messages.push({ role: "user", content: userInput });
      this.messages.push(...this.dueSystemMessages());
      this.turnsSinceReminder += 1;

      for (let turn = 0; turn < MAX_MAIN_TURNS; turn++) {
        const response = await client.messages
          .stream(
            {
              model: this.model,
              max_tokens: 64000,
              system: SYSTEM_PROMPT, // static for the whole session
              output_config: { effort: EFFORT },
              tools: [WORKFLOW_TOOL, BASH_TOOL],
              messages: this.messages,
            },
            { signal: AbortSignal.timeout(REQUEST_TIMEOUT_SECONDS * 1000) },
          )
          .finalMessage();
        this.messages.push({ role: "assistant", content: response.content });

        if (response.stop_reason === "pause_turn") {
          continue;
        }
        if (response.stop_reason !== "tool_use") {
          let text = response.content
            .filter((block): block is Anthropic.TextBlock => block.type === "text")
            .map((block) => block.text)
            .join("");
          if (response.stop_reason === "max_tokens") {
            // 丢弃被截断的助手消息，以免后续轮次在其基础上继续构建。
            this.messages.pop();
            text += "\n\n(warning: response was truncated at max_tokens)";
          }
          return text;
        }

        const toolResults: Anthropic.ToolResultBlockParam[] = [];
        for (const block of response.content) {
          if (block.type !== "tool_use") {
            continue;
          }
          let output: string;
          let isError: boolean;
          if (block.name === "Workflow") {
            const input = block.input as { subtasks?: unknown };
            ({ output, isError } = await runWorkflow(this.model, input.subtasks ?? []));
          } else if (block.name === "bash") {
            ({ output, isError } = await handleBashBlock(block));
          } else {
            output = `unknown tool: ${block.name}`;
            isError = true;
          }
          toolResults.push({
            type: "tool_result",
            tool_use_id: block.id,
            content: output,
            is_error: isError,
          });
        }
        this.messages.push({ role: "user", content: toolResults });
      }
      return "(hit the main loop turn limit before finishing)";
    }
  }
  ```

  ```csharp C#
  // 一个代理循环，其编排模式通过对话中途的系统消息切换。
  List<MessageParam> messages = [];
  var modeOn = true;
  var modeAnnounced = false;
  var exitPending = false;
  var turnsSinceReminder = 0;

  // 开启或关闭该模式。通知将随下一个用户轮次一起发送。
  void SetMode(bool nextModeOn)
  {
      if (nextModeOn == modeOn)
      {
          return;
      }
      if (!nextModeOn)
      {
          if (modeAnnounced)
          {
              exitPending = true;
          }
      }
      else
      {
          exitPending = false;
      }
      modeOn = nextModeOn;
  }

  // Role 属性是开放枚举，因此对话中途的 "system" 角色可以作为
  // 原始字符串赋值；专用常量随 SDK 发布版本一起提供。
  MessageParam SystemMessage(string content) => new() { Role = "system", Content = content };

  // 本轮应发送的系统消息：退出通知、进入时的完整模式文本，
  // 或每隔 turnsBetweenRefreshers 个用户轮次的单行提醒。
  List<MessageParam> DueSystemMessages()
  {
      List<MessageParam> due = [];
      if (exitPending)
      {
          exitPending = false;
          modeAnnounced = false;
          due.Add(SystemMessage(modeExit));
      }
      if (modeOn)
      {
          if (!modeAnnounced)
          {
              modeAnnounced = true;
              turnsSinceReminder = 0;
              due.Add(SystemMessage(modeEnter));
          }
          else if (turnsSinceReminder >= turnsBetweenRefreshers)
          {
              turnsSinceReminder = 0;
              due.Add(SystemMessage(modeRefresh));
          }
      }
      return due;
  }

  // 通过循环发送一个用户轮次，执行工具调用直到模型停止。
  async Task<string> Turn(string userInput)
  {
      // 对话中途的系统消息跟随其所适用的用户轮次之后，这样可以
      // 保持它们之前的缓存前缀不受影响。
      messages.Add(new() { Role = Role.User, Content = userInput });
      messages.AddRange(DueSystemMessages());
      turnsSinceReminder++;

      for (var turn = 0; turn < maxMainTurns; turn++)
      {
          using var deadline = new CancellationTokenSource(TimeSpan.FromSeconds(requestTimeoutSeconds));
          var response = await client.Messages.Create(new MessageCreateParams
          {
              Model = model,
              MaxTokens = requestMaxTokens,
              System = systemPrompt, // static for the whole session
              OutputConfig = new OutputConfig { Effort = effort },
              Tools = [workflowTool, bashTool],
              Messages = messages,
          }, cancellationToken: deadline.Token);
          messages.Add(new()
          {
              Role = Role.Assistant,
              Content = response.Content.Select(block => new ContentBlockParam(block.Json)).ToList(),
          });

          if (response.StopReason == StopReason.PauseTurn)
          {
              continue;
          }
          if (response.StopReason != StopReason.ToolUse)
          {
              var text = string.Concat(
                  response.Content.Select(block => block.TryPickText(out var textBlock) ? textBlock.Text : ""));
              if (response.StopReason == StopReason.MaxTokens)
              {
                  // 丢弃被截断的助手消息，以免下一轮在其基础上继续构建。
                  messages.RemoveAt(messages.Count - 1);
                  text += "\n\n(warning: response was truncated at max_tokens)";
              }
              return text;
          }

          List<ContentBlockParam> toolResults = [];
          foreach (var block in response.Content)
          {
              if (!block.TryPickToolUse(out var toolUse))
              {
                  continue;
              }
              string output;
              bool isError;
              if (toolUse.Name == "Workflow")
              {
                  toolUse.Input.TryGetValue("subtasks", out var rawSubtasks);
                  (output, isError) = await RunWorkflow(rawSubtasks);
              }
              else if (toolUse.Name == "bash")
              {
                  (output, isError) = await HandleBashBlock(toolUse);
              }
              else
              {
                  output = $"unknown tool: {toolUse.Name}";
                  isError = true;
              }
              toolResults.Add(new ToolResultBlockParam(toolUse.ID) { Content = output, IsError = isError });
          }
          messages.Add(new() { Role = Role.User, Content = toolResults });
      }
      return "(hit the main loop turn limit before finishing)";
  }
  ```

  ```go Go
  // modeAgent 是一个代理循环，其编排模式通过对话中途的
  // 系统消息来切换。
  type modeAgent struct {
  	model              string
  	modeOn             bool
  	messages           []anthropic.MessageParam
  	modeAnnounced      bool
  	exitPending        bool
  	turnsSinceReminder int
  }

  func newModeAgent(model string) *modeAgent {
  	return &modeAgent{model: model, modeOn: true}
  }

  // setMode 开启或关闭该模式。通知将随下一个用户轮次一并发送。
  func (agent *modeAgent) setMode(modeOn bool) {
  	if modeOn == agent.modeOn {
  		return
  	}
  	if !modeOn {
  		if agent.modeAnnounced {
  			agent.exitPending = true
  		}
  	} else {
  		agent.exitPending = false
  	}
  	agent.modeOn = modeOn
  }

  // dueSystemMessages 返回本轮应发送的系统消息：退出通知、进入时的
  // 完整模式文本，或每隔 turnsBetweenRefreshers 个用户轮次的单行提醒。
  func (agent *modeAgent) dueSystemMessages() []anthropic.MessageParam {
  	// MessageParamRole 是开放字符串类型，因此对话中途的 "system" 角色可
  	// 直接表达；专用常量将随 SDK 版本一同发布。
  	systemMessage := func(content string) anthropic.MessageParam {
  		return anthropic.MessageParam{
  			Role:    anthropic.MessageParamRole("system"),
  			Content: []anthropic.ContentBlockParamUnion{anthropic.NewTextBlock(content)},
  		}
  	}
  	var due []anthropic.MessageParam
  	if agent.exitPending {
  		agent.exitPending = false
  		agent.modeAnnounced = false
  		due = append(due, systemMessage(modeExit))
  	}
  	if agent.modeOn {
  		if !agent.modeAnnounced {
  			agent.modeAnnounced = true
  			agent.turnsSinceReminder = 0
  			due = append(due, systemMessage(modeEnter))
  		} else if agent.turnsSinceReminder >= turnsBetweenRefreshers {
  			agent.turnsSinceReminder = 0
  			due = append(due, systemMessage(modeRefresh))
  		}
  	}
  	return due
  }

  // turn 将一个用户轮次送入循环，执行工具调用直到模型停止。
  func (agent *modeAgent) turn(ctx context.Context, userInput string) (string, error) {
  	// 对话中途的系统消息跟在其所适用的用户轮次之后，从而保持
  	// 它们之前的缓存前缀不受影响。
  	agent.messages = append(agent.messages, anthropic.NewUserMessage(anthropic.NewTextBlock(userInput)))
  	agent.messages = append(agent.messages, agent.dueSystemMessages()...)
  	agent.turnsSinceReminder++

  	for range maxMainTurns {
  		var response anthropic.Message
  		err := func() error {
  			ctx, cancel := context.WithTimeout(ctx, requestTimeoutSeconds*time.Second)
  			defer cancel()
  			stream := client.Messages.NewStreaming(ctx, anthropic.MessageNewParams{
  				Model:        agent.model,
  				MaxTokens:    64000,
  				System:       []anthropic.TextBlockParam{{Text: systemPrompt}}, // static for the whole session
  				OutputConfig: anthropic.OutputConfigParam{Effort: effort},
  				Tools:        []anthropic.ToolUnionParam{workflowTool, bashTool},
  				Messages:     agent.messages,
  			})
  			defer stream.Close()
  			for stream.Next() {
  				if err := response.Accumulate(stream.Current()); err != nil {
  					return err
  				}
  			}
  			return stream.Err()
  		}()
  		if err != nil {
  			return "", err
  		}
  		agent.messages = append(agent.messages, response.ToParam())

  		if response.StopReason == anthropic.StopReasonPauseTurn {
  			continue
  		}
  		if response.StopReason != anthropic.StopReasonToolUse {
  			var text strings.Builder
  			for _, block := range response.Content {
  				if textBlock, ok := block.AsAny().(anthropic.TextBlock); ok {
  					text.WriteString(textBlock.Text)
  				}
  			}
  			if response.StopReason == anthropic.StopReasonMaxTokens {
  				// 丢弃被截断的助手消息，而不是在历史中留下被裁剪的轮次。
  				agent.messages = agent.messages[:len(agent.messages)-1]
  				text.WriteString("\n\n(warning: response was truncated at max_tokens)")
  			}
  			return text.String(), nil
  		}

  		var toolResults []anthropic.ContentBlockParamUnion
  		for _, block := range response.Content {
  			toolUse, ok := block.AsAny().(anthropic.ToolUseBlock)
  			if !ok {
  				continue
  			}
  			var output string
  			var isError bool
  			switch toolUse.Name {
  			case "Workflow":
  				var input struct {
  					Subtasks json.RawMessage `json:"subtasks"`
  				}
  				if err := json.Unmarshal(toolUse.Input, &input); err != nil {
  					output, isError = fmt.Sprintf("Workflow error: could not parse input: %s", err), true
  				} else {
  					output, isError = runWorkflow(ctx, agent.model, input.Subtasks)
  				}
  			case "bash":
  				output, isError = handleBashBlock(ctx, toolUse)
  			default:
  				output, isError = fmt.Sprintf("unknown tool: %s", toolUse.Name), true
  			}
  			toolResults = append(toolResults, anthropic.NewToolResultBlock(toolUse.ID, output, isError))
  		}
  		agent.messages = append(agent.messages, anthropic.NewUserMessage(toolResults...))
  	}
  	return "(hit the main loop turn limit before finishing)", nil
  }

  ```

  ```java Java
  // 一个代理循环，其编排模式通过对话中途的系统消息切换。
  class ModeAgent {
      private final Model model;
      private boolean modeOn;
      private final List<MessageParam> messages = new ArrayList<>();
      private boolean modeAnnounced = false;
      private boolean exitPending = false;
      private int turnsSinceReminder = 0;

      ModeAgent(Model model) {
          this(model, true);
      }

      ModeAgent(Model model, boolean modeOn) {
          this.model = model;
          this.modeOn = modeOn;
      }

      // 开启或关闭该模式。通知将随下一个用户轮次一并发送。
      void setMode(boolean modeOn) {
          if (modeOn == this.modeOn) {
              return;
          }
          if (!modeOn) {
              if (modeAnnounced) {
                  exitPending = true;
              }
          } else {
              exitPending = false;
          }
          this.modeOn = modeOn;
      }

      // 本轮应发送的系统消息：退出通知、进入时的完整模式文本，
      // 或每隔 TURNS_BETWEEN_REFRESHERS 个用户轮次发送的单行提醒。
      private List<MessageParam> dueSystemMessages() {
          List<MessageParam> due = new ArrayList<>();
          if (exitPending) {
              exitPending = false;
              modeAnnounced = false;
              due.add(systemMessage(MODE_EXIT));
          }
          if (modeOn) {
              if (!modeAnnounced) {
                  modeAnnounced = true;
                  turnsSinceReminder = 0;
                  due.add(systemMessage(MODE_ENTER));
              } else if (turnsSinceReminder >= TURNS_BETWEEN_REFRESHERS) {
                  turnsSinceReminder = 0;
                  due.add(systemMessage(MODE_REFRESH));
              }
          }
          return due;
      }

      // MessageParam.Role 是开放枚举，因此对话中途的 "system" 角色可以
      // 用 Role.of 表示；专用常量将随 SDK 发布版本提供。
      private MessageParam systemMessage(String content) {
          return MessageParam.builder()
                  .role(MessageParam.Role.of("system"))
                  .content(content)
                  .build();
      }

      // 通过循环发送一个用户轮次，执行工具调用直到模型停止。
      String turn(String userInput) throws InterruptedException {
          // 对话中途的系统消息跟随其所适用的用户轮次之后，从而保持
          // 其前面的缓存前缀不受影响。
          messages.add(MessageParam.builder().role(MessageParam.Role.USER).content(userInput).build());
          messages.addAll(dueSystemMessages());
          turnsSinceReminder++;

          for (int turn = 0; turn < MAX_MAIN_TURNS; turn++) {
              MessageCreateParams params = MessageCreateParams.builder()
                      .model(model)
                      .maxTokens(64000L)
                      .system(SYSTEM_PROMPT) // static for the whole session
                      .outputConfig(OutputConfig.builder().effort(EFFORT).build())
                      .addTool(WORKFLOW_TOOL)
                      .addTool(BASH_TOOL)
                      .messages(messages)
                      .build();
              MessageAccumulator accumulator = MessageAccumulator.create();
              try (var stream = client.messages().createStreaming(params, REQUEST_OPTIONS)) {
                  stream.stream().forEach(accumulator::accumulate);
              }
              Message response = accumulator.message();
              messages.add(response.toParam());

              StopReason stopReason = response.stopReason().orElse(null);
              if (StopReason.PAUSE_TURN.equals(stopReason)) {
                  continue;
              }
              if (!StopReason.TOOL_USE.equals(stopReason)) {
                  String text = response.content().stream()
                          .flatMap(block -> block.text().stream())
                          .map(TextBlock::text)
                          .collect(Collectors.joining());
                  if (StopReason.MAX_TOKENS.equals(stopReason)) {
                      // 丢弃被截断的助手消息，以免污染后续轮次。
                      messages.removeLast();
                      text += "\n\n(warning: response was truncated at max_tokens)";
                  }
                  return text;
              }

              List<ContentBlockParam> toolResults = new ArrayList<>();
              for (ContentBlock block : response.content()) {
                  if (block.toolUse().isEmpty()) {
                      continue;
                  }
                  ToolUseBlock toolUse = block.toolUse().get();
                  ToolOutput result = switch (toolUse.name()) {
                      case "Workflow" -> {
                          Map<String, JsonValue> input =
                                  (Map<String, JsonValue>) toolUse._input().asObject().orElse(Map.of());
                          JsonValue rawSubtasks = input.getOrDefault("subtasks", JsonValue.from(List.of()));
                          yield runWorkflow(model, rawSubtasks);
                      }
                      case "bash" -> handleBashBlock(toolUse);
                      default -> new ToolOutput("unknown tool: " + toolUse.name(), true);
                  };
                  toolResults.add(ContentBlockParam.ofToolResult(ToolResultBlockParam.builder()
                          .toolUseId(toolUse.id())
                          .content(result.output())
                          .isError(result.isError())
                          .build()));
              }
              messages.add(MessageParam.builder()
                      .role(MessageParam.Role.USER)
                      .contentOfBlockParams(toolResults)
                      .build());
          }
          return "(hit the main loop turn limit before finishing)";
      }
  }
  ```

  ```php PHP
  /** An agent loop whose orchestration mode is toggled with mid-conversation system messages. */
  class ModeAgent
  {
      private array $messages = [];
      private bool $modeAnnounced = false;
      private bool $exitPending = false;
      private int $turnsSinceReminder = 0;

      public function __construct(
          private readonly Client $client,
          private readonly string $model,
          private bool $modeOn = true,
      ) {
      }

      /** Turn the mode on or off. The notice is delivered with the next user turn. */
      public function setMode(bool $modeOn): void
      {
          if ($modeOn === $this->modeOn) {
              return;
          }
          if ($modeOn) {
              $this->exitPending = false;
          } elseif ($this->modeAnnounced) {
              $this->exitPending = true;
          }
          $this->modeOn = $modeOn;
      }

      public function turn(string $userInput): string
      {
          // 对话中途的系统消息紧跟在其所适用的用户轮次之后，这样可以
          // 保持其前面已缓存的前缀不受影响。
          $this->messages[] = ['role' => 'user', 'content' => $userInput];
          array_push($this->messages, ...$this->dueSystemMessages());
          $this->turnsSinceReminder++;

          for ($turn = 0; $turn < MAX_MAIN_TURNS; $turn++) {
              $stream = $this->client->messages->createStream(
                  model: $this->model,
                  maxTokens: 64000,
                  system: SYSTEM_PROMPT, // static for the whole session
                  outputConfig: ['effort' => EFFORT],
                  tools: [WORKFLOW_TOOL, BASH_TOOL],
                  messages: $this->messages,
                  requestOptions: ['timeout' => REQUEST_TIMEOUT_SECONDS],
              );
              [$content, $stopReason] = drainMessageStream($stream);
              $this->messages[] = ['role' => 'assistant', 'content' => $content];

              if ($stopReason === 'pause_turn') {
                  continue;
              }
              if ($stopReason !== 'tool_use') {
                  $text = '';
                  foreach ($content as $block) {
                      if ($block instanceof TextBlock) {
                          $text .= $block->text;
                      }
                  }
                  if ($stopReason === 'max_tokens') {
                      // 丢弃被截断的助手消息，以免下一轮在其基础上继续构建。
                      array_pop($this->messages);
                      $text .= "\n\n(warning: response was truncated at max_tokens)";
                  }
                  return $text;
              }

              $toolResults = [];
              foreach ($content as $block) {
                  if (!$block instanceof ToolUseBlock) {
                      continue;
                  }
                  if ($block->name === 'Workflow') {
                      [$output, $isError] =
                          runWorkflow($this->client, $this->model, $block->input['subtasks'] ?? []);
                  } elseif ($block->name === 'bash') {
                      [$output, $isError] = handleBashBlock($block);
                  } else {
                      $output = "unknown tool: {$block->name}";
                      $isError = true;
                  }
                  $toolResults[] = [
                      'type' => 'tool_result',
                      'tool_use_id' => $block->id,
                      'content' => $output,
                      'is_error' => $isError,
                  ];
              }
              $this->messages[] = ['role' => 'user', 'content' => $toolResults];
          }
          return '(hit the main loop turn limit before finishing)';
      }

      /**
       * System messages owed on this turn: an exit notice, the full mode text on entry,
       * or a one-line refresher every TURNS_BETWEEN_REFRESHERS user turns.
       */
      private function dueSystemMessages(): array
      {
          $due = [];
          if ($this->exitPending) {
              $this->exitPending = false;
              $this->modeAnnounced = false;
              $due[] = ['role' => 'system', 'content' => MODE_EXIT];
          }
          if ($this->modeOn) {
              if (!$this->modeAnnounced) {
                  $this->modeAnnounced = true;
                  $this->turnsSinceReminder = 0;
                  $due[] = ['role' => 'system', 'content' => MODE_ENTER];
              } elseif ($this->turnsSinceReminder >= TURNS_BETWEEN_REFRESHERS) {
                  $this->turnsSinceReminder = 0;
                  $due[] = ['role' => 'system', 'content' => MODE_REFRESH];
              }
          }
          return $due;
      }
  }
  ```

  ```ruby Ruby
  # 一个代理循环，其编排模式通过对话中途的系统消息来切换。
  class ModeAgent
    def initialize(model, mode_on: true)
      @model = model
      @mode_on = mode_on
      @messages = []
      @mode_announced = false
      @exit_pending = false
      @turns_since_reminder = 0
    end

    # 开启或关闭该模式。通知会随下一个用户轮次一起发送。
    def set_mode(mode_on)
      return if mode_on == @mode_on

      if mode_on
        @exit_pending = false
      else
        @exit_pending = true if @mode_announced
      end
      @mode_on = mode_on
    end

    def turn(user_input)
      # 对话中途的系统消息紧跟在其所适用的用户轮次之后，这样可以保持
      # 它们之前已缓存的前缀不受影响。
      @messages << {role: "user", content: user_input}
      @messages.concat(due_system_messages)
      @turns_since_reminder += 1

      MAX_MAIN_TURNS.times do
        stream = CLIENT.messages.stream(
          model: @model,
          max_tokens: 64_000,
          system_: SYSTEM_PROMPT, # static for the whole session
          output_config: {effort: EFFORT},
          tools: [WORKFLOW_TOOL, BASH_TOOL],
          messages: @messages,
          request_options: {timeout: REQUEST_TIMEOUT_SECONDS}
        )
        response = stream.accumulated_message
        @messages << {role: "assistant", content: assistant_content_param(response.content)}

        next if response.stop_reason == :pause_turn

        unless response.stop_reason == :tool_use
          text = response.content.select { |block| block.type == :text }.map(&:text).join
          if response.stop_reason == :max_tokens
            @messages.pop # drop the truncated assistant message from the history
            text += "\n\n(warning: response was truncated at max_tokens)"
          end
          return text
        end

        tool_results = []
        response.content.each do |block|
          next unless block.type == :tool_use

          input = parse_tool_input(block.input)
          case block.name
          when "Workflow"
            output, is_error = run_workflow(@model, input["subtasks"] || [])
          when "bash"
            output, is_error = handle_bash_block(block)
          else
            output, is_error = "unknown tool: #{block.name}", true
          end
          tool_results << {
            type: "tool_result",
            tool_use_id: block.id,
            content: output,
            is_error: is_error
          }
        end
        @messages << {role: "user", content: tool_results}
      end
      "(hit the main loop turn limit before finishing)"
    end

    private

    # 本轮应发送的系统消息：退出通知、进入时的完整模式文本，
    # 或每隔 TURNS_BETWEEN_REFRESHERS 个用户轮次发送一行提醒。
    def due_system_messages
      due = []
      if @exit_pending
        @exit_pending = false
        @mode_announced = false
        due << {role: "system", content: MODE_EXIT}
      end
      if @mode_on
        if !@mode_announced
          @mode_announced = true
          @turns_since_reminder = 0
          due << {role: "system", content: MODE_ENTER}
        elsif @turns_since_reminder >= TURNS_BETWEEN_REFRESHERS
          @turns_since_reminder = 0
          due << {role: "system", content: MODE_REFRESH}
        end
      end
      due
    end
  end
  ```
</CodeGroup>

## 运行它

<Warning>
  本示例中的 bash 工具在没有沙箱的情况下直接在您的机器上运行模型编写的命令，而扇出会并行运行多个这样的智能体。请在您可以放心暴露的目录和环境中运行它，并在将其用于本地实验之外的任何用途之前添加沙箱。
</Warning>

<CodeGroup>
  ```python Python
  if __name__ == "__main__":
      task = (
          sys.argv[1]
          if len(sys.argv) > 1
          else "Explore the current directory, then give a thorough review: what it does, "
          "code-quality issues, and concrete improvements."
      )
      agent = ModeAgent(MODEL)
      print(agent.turn(task))
      agent.set_mode(False)
      print(agent.turn("Briefly summarize what you found above, no fan-out needed."))
  ```

  ```typescript TypeScript
  const task =
    process.argv[2] ??
    "Explore the current directory, then give a thorough review: what it does, " +
      "code-quality issues, and concrete improvements.";
  const agent = new ModeAgent(MODEL);
  console.log(await agent.turn(task));
  agent.setMode(false);
  console.log(await agent.turn("Briefly summarize what you found above, no fan-out needed."));
  ```

  ```csharp C#
  var task = args.Length > 0
      ? args[0]
      : "Explore the current directory, then give a thorough review: what it does, "
          + "code-quality issues, and concrete improvements.";
  Console.WriteLine(await Turn(task));
  SetMode(false);
  Console.WriteLine(await Turn("Briefly summarize what you found above, no fan-out needed."));
  ```

  ```go Go
  func main() {
  	if err := run(context.Background()); err != nil {
  		log.Fatal(err)
  	}
  }

  func run(ctx context.Context) error {
  	if docTestMode {
  		defer os.RemoveAll(workDir)
  	}
  	task := "Explore the current directory, then give a thorough review: what it does, " +
  		"code-quality issues, and concrete improvements."
  	if len(os.Args) > 1 {
  		task = os.Args[1]
  	}
  	agent := newModeAgent(modelID)
  	answer, err := agent.turn(ctx, task)
  	if err != nil {
  		return err
  	}
  	fmt.Println(answer)

  	agent.setMode(false)
  	summary, err := agent.turn(ctx, "Briefly summarize what you found above, no fan-out needed.")
  	if err != nil {
  		return err
  	}
  	fmt.Println(summary)
  	return nil
  }

  ```

  ```java Java
  void main(String[] args) throws InterruptedException {
      String task = args.length > 0
              ? args[0]
              : "Explore the current directory, then give a thorough review: what it does, "
                      + "code-quality issues, and concrete improvements.";
      ModeAgent agent = new ModeAgent(MODEL);
      IO.println(agent.turn(task));
      agent.setMode(false);
      IO.println(agent.turn("Briefly summarize what you found above, no fan-out needed."));
  }
  ```

  ```php PHP
  $task = $argv[1] ??
      'Explore the current directory, then give a thorough review: what it does, '
      . 'code-quality issues, and concrete improvements.';
  $agent = new ModeAgent($client, MODEL);
  echo $agent->turn($task), PHP_EOL;
  $agent->setMode(false);
  echo $agent->turn('Briefly summarize what you found above, no fan-out needed.'), PHP_EOL;
  ```

  ```ruby Ruby
  task = ARGV[0] ||
    "Explore the current directory, then give a thorough review: what it does, " \
    "code-quality issues, and concrete improvements."
  agent = ModeAgent.new(MODEL)
  puts agent.turn(task)
  agent.set_mode(false)
  puts agent.turn("Briefly summarize what you found above, no fan-out needed.")
  ```
</CodeGroup>

从您希望智能体在其中工作的目录启动示例，例如要审查的代码仓库的根目录：

```bash
python orchestration_mode.py "Review this repository for flaky tests and propose fixes."
```

模式开启后，预期模型会用几条 bash 命令进行侦察，在无需提示的情况下分派 Workflow 工具，并将子智能体报告综合成最终答案。琐碎或闲聊式的请求则按提醒的指示保持单独处理。

## 迈向生产级框架

本示例刻意保持小巧。面向真实工作负载的框架通常会添加：

* **沙箱化的编排脚本：** 让模型输出一个简短的编排程序（分支、循环和归约步骤），并在隔离的解释器中运行它，而不是只接受一个扁平的子任务字符串列表。
* **持久化日志：** 用一个能够在进程重启后存活、并且在跨机器并发写入下安全的存储来替换本地 JSON 文件。
* **预算强制执行：** 跟踪整个会话中启动的子智能体总数，而不仅仅是每次 Workflow 调用的数量，并拒绝超过硬性上限，以免失控的计划耗尽您的配额。

本示例中的模式（模式提醒、工具描述中的持续同意、日志记录和验证波次）可以原样沿用；只有围绕它们的执行基础会变得更加健壮。

## 相关内容

<CardGroup cols={2}>
  <Card title="对话中途系统消息" icon="message" href="https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages">
    模式提醒所使用的机制，以及它如何与提示缓存交互。
  </Card>

  <Card title="Effort" icon="gauge" href="https://platform.claude.com/docs/zh-CN/build-with-claude/effort">
    API 接受的努力级别以及如何选择。
  </Card>

  <Card title="使用 Claude 进行工具使用" icon="wrench" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview">
    定义工具、处理工具调用以及工具结果。
  </Card>

  <Card title="Bash 工具" icon="terminal" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/bash-tool">
    本示例在本地执行的 Anthropic 定义的 bash 工具。
  </Card>
</CardGroup>
