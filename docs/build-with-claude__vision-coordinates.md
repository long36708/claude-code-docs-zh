---
title: 坐标与边界框
url: https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates
description: Claude 如何调整图像大小，以及如何处理它为边界框、点和 UI 元素返回的像素坐标。
---

Claude 可以定位并标注图像中的区域（例如，为表格、表单字段、图表元素或 UI 组件返回边界框）。本指南介绍 Claude 在处理图像之前如何调整图像大小，以及如何处理它返回的像素坐标，以便边界框和点能够与您的原始图像对齐。

在 OCR 流水线、表单提取、图表解析、UI 元素定位，以及任何需要对图像特定区域进行操作的任务中，您都会需要这些内容。有关发送图像、支持的格式以及各模型的分辨率限制，请参阅[视觉](https://platform.claude.com/docs/zh-CN/build-with-claude/vision)。

<Note>
  **Claude 在使用绝对像素坐标时效果最佳。** 请在提示中明确要求使用像素坐标。例如：*"以像素坐标返回每个表格的边界框，格式为 `[x1, y1, x2, y2]`（左上角和右下角）。"* 当您要求归一化坐标时，Claude 的效果不佳，例如：*"返回介于 `0` 和 `1000` 之间的边界框坐标。"* 请始终要求像素坐标，如有需要再在您自己的代码中进行归一化。若要以机器可读的 JSON 而非散文形式获取坐标，请使用[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)定义一个 schema，例如为每个检测到的元素定义一个包含 `[x1, y1, x2, y2]` 数组的对象。
</Note>

坐标遵循标准图像约定：原点 `(0, 0)` 位于图像的左上角，x 向右递增，y 向下递增。Claude 返回的坐标是 Claude 所看到的图像中的像素位置：即 Claude 将您的图像调整大小以适应模型原生分辨率之后的图像（请参阅 [Claude 如何调整图像大小并填充图像](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#how-claude-resizes-and-pads-images)）。要获得可直接使用的坐标，您可以预先调整图像大小，使坐标与您手中的图像一一对应（请参阅[上传前调整图像大小](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#resize-your-image-before-uploading)），或者对 Claude 返回的坐标进行重新缩放（请参阅[无法预先调整大小时重新缩放坐标](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#rescale-coordinates-when-you-cannot-pre-resize)）。

<Note>
  Claude 的空间推理能力存在局限（请参阅[局限性](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#limitations)）。当您在提示中说明预期的坐标格式，并在大规模处理之前对结果进行目视抽查时，坐标精度最佳。图像被缩小时，小元素会损失精度：对于精细目标，请裁剪感兴趣的区域并发送裁剪后的图像（将返回的坐标按裁剪原点进行偏移），或使用高分辨率层级的模型。对于 [PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)，页面会在服务器端以您无法控制的尺寸栅格化为图像，因此返回的坐标无法可靠地映射回页面。若要在 PDF 内容上使用坐标，请自行将页面栅格化为图像，并使用预先调整大小的方法。
</Note>

## Claude 如何调整图像大小并填充图像

Claude 会找出同时满足模型两项图像限制的、保持宽高比的最大尺寸：

1. **边长限制：** 任一边都不超过最大边长（标准层级为 1568 px，高分辨率层级为 2576 px）。
2. **视觉令牌限制：** 图像的令牌成本 `⌈width / 28⌉ × ⌈height / 28⌉` 不超过模型的视觉令牌预算（标准层级为 1568 个令牌，高分辨率层级为 4784 个令牌）。

请参阅[分辨率与令牌成本](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#evaluate-image-size)了解各模型所属的层级。

对于几乎所有照片和屏幕截图，决定最终尺寸的是视觉令牌限制。只有对于全景图或较长的手机截图等狭长图像，边长限制才会起主导作用。请使用[参考实现](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#resize-your-image-before-uploading)计算尺寸，而不要手动按边长缩放：一张 1920×1080 的屏幕截图会被调整为 1456×819，而不是 1568×882，如果假定按边长限制缩放，每个坐标都会明显偏离目标。

即使任一边都未超过边长限制，令牌限制也可能触发调整大小。忽视这一点是坐标错位最常见的原因。例如，一张以 130 DPI 扫描的 A4 页面为 1075×1520 像素：两边都小于 1568 px，但它需要 `39 × 55 = 2145` 个视觉令牌，因此 Claude 会将其调整为 924×1307。

<Note>
  此示例假定使用标准分辨率层级的模型。高分辨率层级的模型不会对同一扫描件调整大小：2145 个令牌在其 4784 个令牌的预算之内，因此它返回的坐标可直接映射到 1075×1520 的原始图像上。模型层级列于[分辨率与令牌成本](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#evaluate-image-size)中。
</Note>

随后，Claude 会对每张图像（无论是否调整过大小）在底边和右边进行填充，直至达到 28 像素的下一个倍数（在本例中 924×1307 变为 924×1316）。填充部分不包含任何内容：Claude 感知到的是填充后的图像，但页面内容始终只占据未填充的调整后区域。**请始终按调整后的尺寸进行归一化或重新缩放，而不是按填充后的尺寸**；除以填充后的尺寸会使每个坐标产生少量缩放偏差。

## 上传前调整图像大小

最可靠的方法是在上传前自行调整图像大小，这样您手中的图像就正是 Claude 所看到的图像，Claude 返回的坐标无需任何转换。

首先确认您的模型属于哪个分辨率层级（请参阅[分辨率与令牌成本](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#evaluate-image-size)），并传入相应的边长和令牌限制。以下参考实现可计算 Claude 将图像调整到的精确尺寸：

<CodeGroup>
  ```bash cURL
  # 此参考实现是本地数学计算，不会发出 API 请求，因此
  # cURL 没有可展示的内容。请参阅 SDK 选项卡。
  ```

  ```bash CLI
  # 此参考实现是本地数学计算，不发起任何 API 请求，因此
  # CLI 没有可展示的内容。请参阅 SDK 选项卡。
  ```

  ```python Python
  import math


  def count_image_tokens(width: int, height: int) -> int:
      """Visual tokens consumed by an image: one token per 28x28 pixel patch."""
      return math.ceil(width / 28) * math.ceil(height / 28)


  def resized_size(
      width: int,
      height: int,
      max_edge: int = 1568,
      max_tokens: int = 1568,
  ) -> tuple[int, int]:
      """The size Claude resizes an image to before padding.

      Defaults are for the standard resolution tier. For high-resolution-tier
      models, use max_edge=2576 and max_tokens=4784. Returns (width, height).
      Images that already fit within the limits are returned unchanged.
      """

      def fits(w: int, h: int) -> bool:
          return (
              math.ceil(w / 28) * 28 <= max_edge
              and math.ceil(h / 28) * 28 <= max_edge
              and count_image_tokens(w, h) <= max_tokens
          )

      if fits(width, height):
          return (width, height)
      if height > width:
          resized_h, resized_w = resized_size(height, width, max_edge, max_tokens)
          return (resized_w, resized_h)

      # 沿长边进行二分搜索，找出保持宽高比且能容纳的
      # 最大尺寸。
      aspect_ratio = width / height
      lo, hi = 1, width  # lo always fits; hi never fits
      while lo + 1 < hi:
          mid = (lo + hi) // 2
          if fits(mid, max(round(mid / aspect_ratio), 1)):
              lo = mid
          else:
              hi = mid
      return (lo, max(round(lo / aspect_ratio), 1))


  # 来自"Claude 如何调整图像尺寸并填充"一节的 A4 示例：
  print(resized_size(1075, 1520))  # (924, 1307)

  # 要应用尺寸调整，请使用您的图像库，例如 Pillow：
  # image.resize(resized_size(*image.size))
  ```

  ```typescript TypeScript
  /** Visual tokens consumed by an image: one token per 28x28 pixel patch. */
  function countImageTokens(width: number, height: number): number {
    return Math.ceil(width / 28) * Math.ceil(height / 28);
  }

  /**
   * Round half to even (banker's rounding), matching Python's round(). The
   * live API resolves exact .5 ties toward the even neighbor, so Math.round
   * (which rounds halves up) would compute a different size for some images.
   */
  function roundTiesToEven(value: number): number {
    const floor = Math.floor(value);
    if (value - floor !== 0.5) return Math.round(value);
    return floor % 2 === 0 ? floor : floor + 1;
  }

  /**
   * The size Claude resizes an image to before padding.
   *
   * Defaults are for the standard resolution tier. For high-resolution-tier
   * models, use maxEdge = 2576 and maxTokens = 4784. Returns [width, height].
   * Images that already fit within the limits are returned unchanged.
   */
  function resizedSize(
    width: number,
    height: number,
    maxEdge = 1568,
    maxTokens = 1568
  ): [number, number] {
    const fits = (w: number, h: number): boolean =>
      Math.ceil(w / 28) * 28 <= maxEdge &&
      Math.ceil(h / 28) * 28 <= maxEdge &&
      countImageTokens(w, h) <= maxTokens;

    if (fits(width, height)) return [width, height];
    if (height > width) {
      const [resizedH, resizedW] = resizedSize(height, width, maxEdge, maxTokens);
      return [resizedW, resizedH];
    }

    // 沿长边进行二分搜索，找出保持宽高比且能容纳的
    // 最大尺寸。
    const aspectRatio = width / height;
    let lo = 1; // lo always fits
    let hi = width; // hi never fits
    while (lo + 1 < hi) {
      const mid = Math.floor((lo + hi) / 2);
      if (fits(mid, Math.max(roundTiesToEven(mid / aspectRatio), 1))) {
        lo = mid;
      } else {
        hi = mid;
      }
    }
    return [lo, Math.max(roundTiesToEven(lo / aspectRatio), 1)];
  }

  // 来自“Claude 如何调整图像尺寸并填充”的 A4 示例：
  console.log(resizedSize(1075, 1520)); // [ 924, 1307 ]

  // 要应用尺寸调整，请使用您的图像库，例如 sharp：
  // await sharp(input).resize(width, height).toBuffer()
  ```

  ```csharp C#
  // 图像消耗的视觉令牌数：每个 28x28 像素块对应一个令牌。
  static int CountImageTokens(int width, int height)
  {
      return (width + 27) / 28 * ((height + 27) / 28); // ceil(w/28) * ceil(h/28)
  }

  // Claude 在填充前将图像调整到的尺寸。默认值适用于
  // 标准分辨率层级；对于高分辨率层级模型，请传入
  // maxEdge: 2576, maxTokens: 4784。已符合限制的图像
  // 将原样返回。
  static (int Width, int Height) ResizedSize(
      int width, int height, int maxEdge = 1568, int maxTokens = 1568)
  {
      bool Fits(int w, int h) =>
          (w + 27) / 28 * 28 <= maxEdge
          && (h + 27) / 28 * 28 <= maxEdge
          && CountImageTokens(w, h) <= maxTokens;

      if (Fits(width, height))
      {
          return (width, height);
      }
      if (height > width)
      {
          (int resizedH, int resizedW) = ResizedSize(height, width, maxEdge, maxTokens);
          return (resizedW, resizedH);
      }

      // 沿长边进行二分搜索，寻找保持宽高比且能容纳的
      // 最大尺寸。短边采用四舍六入五成双，与实时 API 在
      // 恰好 .5 时的行为一致（MidpointRounding.ToEven，即 Math.Round 的默认值）。
      double aspectRatio = (double)width / height;
      int lo = 1; // lo always fits
      int hi = width; // hi never fits
      while (lo + 1 < hi)
      {
          int mid = (lo + hi) / 2;
          if (Fits(mid, ShortEdge(mid)))
          {
              lo = mid;
          }
          else
          {
              hi = mid;
          }
      }
      return (lo, ShortEdge(lo));

      int ShortEdge(int longEdge) =>
          Math.Max((int)Math.Round(longEdge / aspectRatio, MidpointRounding.ToEven), 1);
  }

  // 来自"Claude 如何调整图像尺寸并填充"的 A4 示例：
  Console.WriteLine(ResizedSize(1075, 1520)); // (924, 1307)
  ```

  ```go Go
  // countImageTokens 计算图像消耗的视觉令牌数：每个
  // 28x28 像素块对应一个令牌。
  func countImageTokens(width, height int) int {
  	return ((width + 27) / 28) * ((height + 27) / 28) // ceil(w/28) * ceil(h/28)
  }

  // resizedSize 返回 Claude 在填充前将图像调整到的尺寸，
  // 格式为 (width, height)。标准分辨率层级传入 maxEdge 1568 和 maxTokens 1568，
  // 高分辨率层级传入 2576 和 4784。已符合
  // 限制的图像将原样返回。
  // "Claude 如何调整图像尺寸并填充"中的 A4 示例：
  // resizedSize(1075, 1520, 1568, 1568) 返回 (924, 1307)。
  func resizedSize(width, height, maxEdge, maxTokens int) (int, int) {
  	fits := func(w, h int) bool {
  		return ((w+27)/28)*28 <= maxEdge &&
  			((h+27)/28)*28 <= maxEdge &&
  			countImageTokens(w, h) <= maxTokens
  	}

  	if fits(width, height) {
  		return width, height
  	}
  	if height > width {
  		resizedH, resizedW := resizedSize(height, width, maxEdge, maxTokens)
  		return resizedW, resizedH
  	}

  	// 沿长边进行二分搜索，寻找保持宽高比且符合限制的
  	// 最大尺寸。短边采用四舍六入五成双（math.RoundToEven），
  	// 与实时 API 在恰好 .5 时的行为一致；math.Round 会将其向上舍入。
  	aspectRatio := float64(width) / float64(height)
  	lo, hi := 1, width // lo always fits; hi never fits
  	for lo+1 < hi {
  		mid := (lo + hi) / 2
  		short := max(int(math.RoundToEven(float64(mid)/aspectRatio)), 1)
  		if fits(mid, short) {
  			lo = mid
  		} else {
  			hi = mid
  		}
  	}
  	return lo, max(int(math.RoundToEven(float64(lo)/aspectRatio)), 1)
  }

  ```

  ```java Java
  /** A resized image size, as returned by resizedSize. */
  record Size(int width, int height) {}

  /** Visual tokens consumed by an image: one token per 28x28 pixel patch. */
  static int countImageTokens(int width, int height) {
      return Math.ceilDiv(width, 28) * Math.ceilDiv(height, 28);
  }

  /**
   * The size Claude resizes an image to before padding.
   *
   * <p>Pass maxEdge 1568 and maxTokens 1568 for the standard resolution tier,
   * or 2576 and 4784 for the high-resolution tier. Images that already fit
   * within the limits are returned unchanged.
   *
   * <p>The A4 example from "How Claude resizes and pads images":
   * resizedSize(1075, 1520, 1568, 1568) returns new Size(924, 1307).
   */
  static Size resizedSize(int width, int height, int maxEdge, int maxTokens) {
      if (fits(width, height, maxEdge, maxTokens)) {
          return new Size(width, height);
      }
      if (height > width) {
          Size rotated = resizedSize(height, width, maxEdge, maxTokens);
          return new Size(rotated.height(), rotated.width());
      }

      // 沿长边进行二分搜索，寻找保持宽高比且能容纳的
      // 最大尺寸。短边采用四舍六入五成双（Math.rint），
      // 与实时 API 在恰好 .5 时的行为一致；Math.round 则会向上舍入。
      double aspectRatio = (double) width / height;
      int lo = 1; // lo always fits
      int hi = width; // hi never fits
      while (lo + 1 < hi) {
          int mid = (lo + hi) / 2;
          if (fits(mid, shortEdge(mid, aspectRatio), maxEdge, maxTokens)) {
              lo = mid;
          } else {
              hi = mid;
          }
      }
      return new Size(lo, shortEdge(lo, aspectRatio));
  }

  private static boolean fits(int width, int height, int maxEdge, int maxTokens) {
      return Math.ceilDiv(width, 28) * 28 <= maxEdge
              && Math.ceilDiv(height, 28) * 28 <= maxEdge
              && countImageTokens(width, height) <= maxTokens;
  }

  private static int shortEdge(int longEdge, double aspectRatio) {
      return Math.max((int) Math.rint(longEdge / aspectRatio), 1);
  }
  ```

  ```php PHP
  // 图像消耗的视觉令牌数：每个 28x28 像素块对应一个令牌。
  function countImageTokens(int $width, int $height): int
  {
      return intdiv($width + 27, 28) * intdiv($height + 27, 28);
  }

  /**
   * The size Claude resizes an image to before padding, as [width, height].
   *
   * Defaults are for the standard resolution tier. For high-resolution-tier
   * models, pass maxEdge: 2576, maxTokens: 4784. Images that already fit
   * within the limits are returned unchanged.
   */
  function resizedSize(int $width, int $height, int $maxEdge = 1568, int $maxTokens = 1568): array
  {
      $fits = fn (int $w, int $h): bool =>
          intdiv($w + 27, 28) * 28 <= $maxEdge
          && intdiv($h + 27, 28) * 28 <= $maxEdge
          && countImageTokens($w, $h) <= $maxTokens;

      if ($fits($width, $height)) {
          return [$width, $height];
      }
      if ($height > $width) {
          [$resizedH, $resizedW] = resizedSize($height, $width, $maxEdge, $maxTokens);
          return [$resizedW, $resizedH];
      }

      // 沿长边二分搜索能容纳的最大保持宽高比的
      // 尺寸。短边采用四舍六入五成双舍入
      // （PHP_ROUND_HALF_EVEN），在恰好 .5 的平局时与实时 API 一致。
      $aspectRatio = $width / $height;
      $lo = 1; // lo always fits
      $hi = $width; // hi never fits
      while ($lo + 1 < $hi) {
          $mid = intdiv($lo + $hi, 2);
          $short = max((int) round($mid / $aspectRatio, 0, PHP_ROUND_HALF_EVEN), 1);
          if ($fits($mid, $short)) {
              $lo = $mid;
          } else {
              $hi = $mid;
          }
      }

      return [$lo, max((int) round($lo / $aspectRatio, 0, PHP_ROUND_HALF_EVEN), 1)];
  }

  // "Claude 如何缩放和填充图像"中的 A4 示例：
  [$resizedWidth, $resizedHeight] = resizedSize(1075, 1520);
  echo "({$resizedWidth}, {$resizedHeight})\n"; // (924, 1307)
  ```

  ```ruby Ruby
  # 图像消耗的视觉令牌数：每个 28x28 像素块对应一个令牌。
  def count_image_tokens(width, height)
    width.ceildiv(28) * height.ceildiv(28)
  end

  # Claude 在填充前将图像调整到的尺寸，格式为 [width, height]。
  #
  # 默认值适用于标准分辨率层级。对于高分辨率层级的
  # 模型，请传入 max_edge: 2576, max_tokens: 4784。已符合
  # 限制的图像将原样返回。
  def resized_size(width, height, max_edge = 1568, max_tokens = 1568)
    fits = lambda do |w, h|
      w.ceildiv(28) * 28 <= max_edge &&
        h.ceildiv(28) * 28 <= max_edge &&
        count_image_tokens(w, h) <= max_tokens
    end

    return [width, height] if fits.call(width, height)

    if height > width
      resized_h, resized_w = resized_size(height, width, max_edge, max_tokens)
      return [resized_w, resized_h]
    end

    # 沿长边进行二分搜索，找出保持宽高比且能容纳的
    # 最大尺寸。短边采用四舍六入五成双（round(half: :even)），
    # 与实时 API 在恰好 .5 的情况下保持一致。
    aspect_ratio = width.fdiv(height)
    lo = 1      # lo always fits
    hi = width  # hi never fits
    while lo + 1 < hi
      mid = (lo + hi) / 2
      short = [(mid / aspect_ratio).round(half: :even), 1].max
      if fits.call(mid, short)
        lo = mid
      else
        hi = mid
      end
    end

    [lo, [(lo / aspect_ratio).round(half: :even), 1].max]
  end

  # 来自“Claude 如何调整图像尺寸并填充”的 A4 示例：
  p resized_size(1075, 1520) # => [924, 1307]
  ```
</CodeGroup>

1. 将图像调整为调整大小辅助函数返回的尺寸。如果图像已经在模型的限制范围内，辅助函数会原样返回其尺寸，无需调整大小。
2. [将调整后的图像发送](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#send-images-to-claude)到 API。不要自行填充。Claude 会处理填充，而且填充不会移动坐标原点。
3. 在提示中明确要求像素坐标。例如：*"以像素坐标返回 Submit 按钮的点击点，格式为 `[x, y]`。"*
4. 直接将返回的坐标用于您发送的图像。如果需要归一化坐标，请除以您发送的图像的尺寸，而不是原始图像的尺寸，也不是填充后的尺寸。

<Note>
  [令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)端点根据图像尺寸估算其令牌成本，而不会完整处理图像，因此计数成功并不意味着图像在 Messages API 的[请求限制](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#request-limits)之内。图像可能计数成功，但在您发送时仍被拒绝。
</Note>

## 使用 `transformations` 将调整大小转为错误

预先调整大小只有在您的流水线持续生成正确尺寸时才能保护您的坐标。新的图像来源或切换到不同分辨率层级的模型，都可能悄然重新引入服务器端的调整大小。要将这种无声的偏移转变为可见的错误，请在 [Messages](https://platform.claude.com/docs/zh-CN/api/messages) 请求中的图像内容块上设置可选的 `transformations` 字段：

```json
{
  "type": "image",
  "source": { "type": "base64", "media_type": "image/png", "data": "..." },
  "transformations": { "oversized_image": "error" }
}
```

如果请求中被标记的图像（任何设置了 `"oversized_image": "error"` 的块）会被调整大小，该请求将被拒绝并返回 400 `invalid_request_error`，其中会指明图像的尺寸以及可容纳的最大尺寸。图像是否触发拒绝取决于[请求中指定的每个模型的限制](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#how-claude-resizes-and-pads-images)：下面的 1920×1080 示例会被标准层级模型拒绝，但在高分辨率层级的范围之内：

```text wrap
messages.0.content.0: image dimensions 1920x1080 exceed the maximum image size of a model named on this request and would be downsized to 1456x819; scale the image to at most 1456x819 or set the image's oversized_image setting to "downsize"
```

请按报告的目标尺寸重新缩放并重新发送：该目标是在您图像的宽高比下，请求中指定的每个模型都能接受的最大尺寸。被标记的图像与[服务器端回退 beta](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback) 的交互方式在该功能的说明中有所描述；在任何模式下，被标记的图像都绝不会以调整大小后的形式提供。

该设置是按图像生效的。`"oversized_image": "downsize"`（省略该字段时的默认值）会保留本页所述的自动调整大小行为。每个图像块仅根据其自身的设置进行检查，因此一个请求中可以混合尺寸至关重要的图像（您将要点击的屏幕截图）和调整大小无害的图像（徽标）。该设置会改变和不会改变的内容如下：

* 填充（[绝不会丢弃内容](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#how-claude-resizes-and-pads-images)）、格式转换和方向校正照常进行。
* 硬性限制（最长边 8000 px，以及[多图像请求](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#request-limits)中更严格的单图像限制）属于单独的拒绝情形；此设置绝不会让图像绕过这些限制。
* 通过 URL 或文件 ID 提供的图像会在其字节被获取后进行检查；这些拒绝携带相同的消息，但没有开头的位置信息，因此无法指明是哪张图像失败；只有嵌入的 base64 图像会在错误中按位置指明。
* [PDF 页面](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)在服务器端以您无法控制的尺寸栅格化；`document` 块不接受该字段（嵌套在文档内容中的图像块则像其他图像块一样接受该字段）。
* 无法确定尺寸的被标记图像会被拒绝，而不是被放行：该拒绝会报告无法确定图像的源尺寸，而不是上文引用的调整大小消息。任何设置了 `"error"` 的图像都不会以调整大小后的形式到达模型。

[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)端点同样遵循 `transformations`，会与 Messages API 完全一致地拒绝嵌入图像，因此您可以在运行推理之前检查嵌入图像是否能在不被调整大小的情况下容纳。计数从不获取通过 URL 或文件 ID 提供的图像，因此来自这些来源的被标记图像仅在 Messages 调用时进行检查，如上所述。

## 无法预先调整大小时重新缩放坐标

如果您无法预先调整大小（例如，图像来自您无法修改的上游系统），请使用[上传前调整图像大小](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#resize-your-image-before-uploading)中的调整大小辅助函数来还原 Claude 所看到的尺寸，然后将 Claude 返回的坐标映射为归一化坐标或映射回您的原始图像。除非图像[选择改为报错](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#oversized-image-error)，否则 Claude 会对超大图像进行调整大小而不是拒绝，直至达到 API 的[请求限制](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#request-limits)。超出这些限制时，请求会以验证错误失败。请传入与您所调用模型相匹配的层级限制：错误层级的限制会还原出错误的调整后尺寸，并悄然使每个坐标发生偏移。此方法需要知道您上传的图像的像素尺寸，因此不适用于 PDF 上传。

您返回给[计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool#handle-coordinate-scaling-for-higher-resolutions)和[浏览器使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool#targets-and-coordinates)工具集的屏幕截图和缩放图像是自动调整大小的例外。对于超出模型限制的 `tool_result` 图像，API 会以验证错误拒绝，而不是对其调整大小。请在返回这些图像之前在您的应用程序中调整其大小，然后将 Claude 返回的坐标缩放回您屏幕的尺寸。

<CodeGroup>
  ```bash cURL
  # 此本地坐标转换不会发出 API 请求，因此
  # cURL 无可展示的内容。请参阅 SDK 选项卡。
  ```

  ```bash CLI
  # 此本地坐标转换不会发出 API 请求，因此
  # CLI 没有可展示的内容。请参阅 SDK 选项卡。
  ```

  ```python Python
  # 此辅助函数调用本页尺寸调整示例中的 resized_size。
  def to_relative_coordinates(
      x: float,
      y: float,
      original_width: int,
      original_height: int,
      max_edge: int = 1568,
      max_tokens: int = 1568,
  ) -> tuple[float, float]:
      """Map a pixel coordinate returned by Claude to relative coordinates in [0, 1].

      Pass the dimensions of the image you uploaded. For high-resolution-tier
      models, use max_edge=2576 and max_tokens=4784.
      """
      resized_w, resized_h = resized_size(
          original_width, original_height, max_edge, max_tokens
      )
      return (x / resized_w, y / resized_h)


  # Claude 在调整后的 A4 页面上返回的位于 (462, 653.5) 的表格角点
  # 映射回 1075x1520 原图的方式如下：
  rel_x, rel_y = to_relative_coordinates(462, 653.5, 1075, 1520)
  print((rel_x * 1075, rel_y * 1520))  # (537.5, 760.0)
  ```

  ```typescript TypeScript
  // 此辅助函数调用本页尺寸调整示例中的 resizedSize。
  /**
   * Map a pixel coordinate returned by Claude to relative coordinates in [0, 1].
   *
   * Pass the dimensions of the image you uploaded. For high-resolution-tier
   * models, use maxEdge = 2576 and maxTokens = 4784.
   */
  function toRelativeCoordinates(
    x: number,
    y: number,
    originalWidth: number,
    originalHeight: number,
    maxEdge = 1568,
    maxTokens = 1568
  ): [number, number] {
    const [resizedW, resizedH] = resizedSize(
      originalWidth,
      originalHeight,
      maxEdge,
      maxTokens
    );
    return [x / resizedW, y / resizedH];
  }

  // Claude 在调整尺寸后的 A4 页面上返回的位于 (462, 653.5) 的表格角点
  // 映射回 1075x1520 原图的方式如下：
  const [relX, relY] = toRelativeCoordinates(462, 653.5, 1075, 1520);
  console.log([relX * 1075, relY * 1520]); // [ 537.5, 760 ]
  ```

  ```csharp C#
  // 此辅助函数调用本页尺寸调整示例中的 ResizedSize。
  // 将 Claude 返回的像素坐标映射为 [0, 1] 范围内的
  // 相对坐标。请传入您上传图像的尺寸，以及与
  // ResizedSize 相同的层级限制。
  static (double X, double Y) ToRelativeCoordinates(
      double x, double y, int originalWidth, int originalHeight,
      int maxEdge = 1568, int maxTokens = 1568)
  {
      (int resizedW, int resizedH) =
          ResizedSize(originalWidth, originalHeight, maxEdge, maxTokens);
      return (x / resizedW, y / resizedH);
  }

  // Claude 在调整尺寸后的 A4 页面上返回位于 (462, 653.5) 的表格角点，
  // 映射回 1075x1520 原图的方式如下：
  (double relX, double relY) = ToRelativeCoordinates(462, 653.5, 1075, 1520);
  Console.WriteLine((relX * 1075, relY * 1520)); // (537.5, 760)
  ```

  ```go Go
  // 此辅助函数调用本页尺寸调整示例中的 resizedSize。

  // toRelativeCoordinates 将 Claude 返回的像素坐标映射为
  // [0, 1] 范围内的相对坐标。请传入您上传图像的尺寸，
  // 以及与 resizedSize 相同的层级限制。
  func toRelativeCoordinates(
  	x, y float64,
  	originalWidth, originalHeight, maxEdge, maxTokens int,
  ) (float64, float64) {
  	resizedW, resizedH := resizedSize(originalWidth, originalHeight, maxEdge, maxTokens)
  	return x / float64(resizedW), y / float64(resizedH)
  }

  // 要映射回原始图像的像素空间，请乘以原始尺寸：
  // 在调整后的 A4 页面上返回于 (462, 653.5) 的表格角点，
  // 在 1075x1520 原图上为 (relX*1075, relY*1520) = (537.5, 760)。
  ```

  ```java Java
  // 此辅助函数调用本页缩放示例中的 resizedSize。
  /** A coordinate scaled into the [0, 1] range on both axes. */
  record RelativeCoordinate(double x, double y) {}

  /**
   * Map a pixel coordinate returned by Claude to relative coordinates in
   * [0, 1]. Pass the dimensions of the image you uploaded, and the same tier
   * limits used for resizedSize.
   */
  static RelativeCoordinate toRelativeCoordinates(
          double x, double y, int originalWidth, int originalHeight, int maxEdge, int maxTokens) {
      Size resized = resizedSize(originalWidth, originalHeight, maxEdge, maxTokens);
      return new RelativeCoordinate(x / resized.width(), y / resized.height());
  }

  // 要映射回原始图像的像素空间，请乘以原始尺寸：
  // 在缩放后的 A4 页面上返回于 (462, 653.5) 的表格角点，
  // 在 1075x1520 原图上即为 (relative.x() * 1075, relative.y() * 1520)
  // = (537.5, 760)。
  ```

  ```php PHP
  // 此辅助函数调用本页缩放示例中的 resizedSize()。
  /**
   * Map a pixel coordinate returned by Claude to relative coordinates in
   * [0, 1], as [x, y]. Pass the dimensions of the image you uploaded, and the
   * same tier limits used for resizedSize.
   */
  function toRelativeCoordinates(
      float $x,
      float $y,
      int $originalWidth,
      int $originalHeight,
      int $maxEdge = 1568,
      int $maxTokens = 1568,
  ): array {
      [$resizedW, $resizedH] = resizedSize($originalWidth, $originalHeight, $maxEdge, $maxTokens);

      return [$x / $resizedW, $y / $resizedH];
  }

  // Claude 在缩放后的 A4 页面上返回的位于 (462, 653.5) 的表格角点
  // 映射回 1075x1520 原图的方式如下：
  [$relX, $relY] = toRelativeCoordinates(462, 653.5, 1075, 1520);
  echo '(' . $relX * 1075 . ', ' . $relY * 1520 . ")\n"; // (537.5, 760)
  ```

  ```ruby Ruby
  # 此辅助函数调用本页尺寸调整示例中的 resized_size。
  # 将 Claude 返回的像素坐标映射为 [0, 1] 范围内的相对坐标，
  # 格式为 [x, y]。请传入您上传的图像的尺寸，以及
  # 与 resized_size 相同的层级限制。
  def to_relative_coordinates(
    x, y, original_width, original_height, max_edge = 1568, max_tokens = 1568
  )
    resized_w, resized_h = resized_size(original_width, original_height, max_edge, max_tokens)
    [x.fdiv(resized_w), y.fdiv(resized_h)]
  end

  # Claude 在调整后的 A4 页面上返回的位于 (462, 653.5) 的表格角点
  # 映射回 1075x1520 原图的方式如下：
  rel_x, rel_y = to_relative_coordinates(462, 653.5, 1075, 1520)
  p [rel_x * 1075, rel_y * 1520] # => [537.5, 760.0]
  ```
</CodeGroup>

填充仅应用于底边和右边，因此原点不会移动，按轴进行线性重新缩放即可。在重新缩放之前，请将返回的坐标钳制到调整后的尺寸范围内，以免略微超出图像的点映射到原始图像之外。

相对坐标可与您所操作的任何表面相乘：原始图像、全分辨率扫描件或屏幕。当您在屏幕上操作且屏幕截图像素与逻辑坐标不同（HiDPI 显示器）时，还需除以显示缩放因子。[计算机使用工具的缩放指南](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool#handle-coordinate-scaling-for-higher-resolutions)涵盖了这种模式。

## 后续步骤

<CardGroup cols={2}>
  <Card title="Agent Skills" icon="stack" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview">
    Agent Skills 是扩展 Claude 功能的模块化能力。每个 Skill 都打包了指令、元数据和可选资源（脚本、模板），Claude 会在相关时自动使用它们。
  </Card>

  <Card title="计算机使用工具" icon="computer" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool">
    通过计算机使用工具，让 Claude 获得桌面环境的屏幕截图、鼠标和键盘控制能力。
  </Card>

  <Card title="PDF 支持" icon="file" href="https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support">
    使用 Claude 处理 PDF。从您的文档中提取文本、分析图表并理解视觉内容。
  </Card>

  <Card title="令牌计数" icon="calculator" href="https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting">
    在将消息发送给 Claude 之前计算其中的令牌数。使用令牌计数来管理速率限制和成本、做出模型路由决策，并使提示符合目标长度。
  </Card>
</CardGroup>
