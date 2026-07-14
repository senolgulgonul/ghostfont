# Ghost Font

Text hidden in motion. Invisible in every frame, readable in the flow.

**Live tool: https://senolgulgonul.github.io/ghostfont**

## What this is

Ghost Font is a visual encoding that hides text in a field of moving dots. The letters are drawn with dots that stay still while the background dots flow past them. Pause the video on any single frame and the text vanishes: statistically, a letter dot and a background dot are identical. Play the video and human motion perception separates them instantly.

The encoding circulated with the claim that only a human can read it and that AI models cannot. This repository is the counterexample. The tool decodes the hidden text from the video, outputs it as an image, and reads it out as a string with OCR. On the original clip it recovers the exact sentence.

## Why the claim fails

The claim is true in a narrower form than intended: no reader restricted to a single frame can extract the text. A stopped frame is statistically uniform, and we verified that per-pixel temporal minimum, maximum, and mean all carry no signal.

But a human watching the video uses many frames, so the fair machine comparison also gets many frames. With the temporal axis available, the message is carried by motion coherence, and motion coherence is a first-order statistic. The same cue that makes the text human-readable makes it machine-readable.

Chat models failing on a pasted video mostly shows that the product did not feed them the frames, not that the encoding is safe. Given frames, the decode is a few seconds of arithmetic.

## How the decoder works

1. **Frame capture.** Every presented frame is grabbed via `requestVideoFrameCallback` (seek-based fallback at 30 fps), converted to grayscale, width capped for speed.
2. **Global motion estimation.** For each consecutive frame pair, the background shift is found by an integer SAD search, then refined to subpixel accuracy with a parabolic fit on the SAD curve. Subpixel matters: the original clip flows at 2.4 px/frame, and integer compensation leaves residue everywhere.
3. **Motion-compensated residual.** The previous frame is warped by the estimated shift and subtracted. Dots riding the flow cancel; the static text dots do not, and survive as high residual. No spatial window, so glyph edges stay sharp.
4. **Temporal windows.** Residuals are accumulated over short overlapping windows (default 20 frames) instead of the whole clip, because the hidden text itself drifts slowly and a full-clip average ghosts. The sharpest window is selected by contrast; all windows are shown as clickable thumbnails.
5. **OCR.** The selected map is de-ghosted (echo offset estimated by autocorrelation of the row profile, attenuated shifted copy subtracted), Otsu-binarized, optionally stroke-thinned, upscaled, and passed to Tesseract at two page-segmentation modes across the top three windows. A confidence-weighted vote over the candidate strings picks the answer.

A fast frame-difference mode is included as a preview. It reveals the text to the eye without any motion model, but the empty lanes of the dot lattice also produce near-zero difference and masquerade as text, so OCR fails on it. Compensation asks the better question: not "did this pixel change" but "did it change the way the global flow predicts".

## Methods that do not work

Tested and rejected, kept here so nobody repeats them:

| Method | Result |
|---|---|
| Temporal min / max / mean | No signal. The encoding is purely temporal. |
| Dot trails (union of dark pixels) | Floods black. The flow runs behind the glyphs too. |
| Dark-persistence counting | Faint signal, below OCR threshold. Moving dots dwell ~2 frames, drifting static dots not much longer. |
| Plain frame differencing | Human-readable, OCR fails. Lattice gaps mimic text. |
| Sobel / unsharp before OCR | Sharpens the ghost as much as the text. |
| Registering and averaging window maps | Text drift is not a pure translation; the average smears. |

## Usage

Open `index.html` in a browser, or use the live page. No build, no server, no dependencies. Select a video file, press Decode, optionally press Read text. Everything runs locally in the browser; the video never leaves the machine. The only network access is the Tesseract.js CDN fetch when the OCR button is pressed.

If the decoded map shows a vertical echo of the text, the text is drifting fast relative to the window: shorten the window. If the map is speckled with broken strokes, the dots are faint or sparse: lengthen it. The usable range is a wide plateau, roughly 3x, not a knife edge.

## The honest conclusion

Ghost Font does not separate humans from machines. It separates single-frame readers from multi-frame readers, and every reader can afford the second frame. The perceptual trick is genuinely elegant, and the asymmetry is real: humans read it for free, pre-attentively, while a machine needs about eighty lines of JavaScript. That is a CAPTCHA-style speed bump, not a firewall. Anything perceptible to humans is, by that very fact, computable.

## License

MIT
