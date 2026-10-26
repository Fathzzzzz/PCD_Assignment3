# PCD Assignment 03: Segmentation and Morphological Processing

## Identity

| | |
|---|---|
| **Name** | Muhammad Fathan Romiza |
| **Student ID** | 25/560518/PA/23623 |
| **University** | Universitas Gadjah Mada |
| **Course** | Digital Image Processing (Pengolahan Citra Digital) |

## About This Project

Third assignment for the Digital Image Processing course. The task is to segment an image with Otsu thresholding and then run the four basic morphological operations on the binary result, with every operator written from scratch instead of called from a library.

Six images are used: a flame on a dark background, handwriting on white paper, a traffic photo, an aerial shot of a tanker, a satellite view, and a flat shape image. The set was picked so that the same fixed pipeline has to survive both dark and bright pictures, and the results show where that assumption breaks.

## Objective

- Convert to grayscale with the luma weights, without `cv2.cvtColor`.
- Build a 5x5 Gaussian kernel and apply it through a convolution loop written by hand.
- Implement Otsu's method from the histogram: find the threshold that maximizes between-class variance.
- Implement erosion, dilation, opening, and closing on a binary image with a 3x3 structuring element.
- Run one fixed pipeline on all six images and compare what each operation does to each kind of content.

## Theoretical Basis

### Grayscale conversion

Luma from the three channels: `gray = 0.299·R + 0.587·G + 0.114·B`, then cast to `uint8`. The weights come from Rec. 601 and match the `RGB2GRAY` conversion used in OpenCV.

### Gaussian blur

`convolve()` multiplies a 5x5 window around each pixel by the kernel and sums it, with `np.pad(..., mode='edge')` extending the border. The kernel itself is

```
G(x, y) = exp( -(x² + y²) / (2σ²) ) / sum
```

with `size = 5`, `σ = 1`, normalized so the weights add up to 1.

### Otsu thresholding

For every candidate threshold `t`, the histogram is split into a below class `B` and an above class `F`. The criterion maximized over `t` is the between-class variance

```
ω = (wB · wF) / total²
σ²_B(t) = ω · (μ_B − μ_F)²
```

The implementation keeps the counts and running sums, computes `μ_B` and `μ_F` at each `t`, and returns the `t` with the largest value. It maximizes `wB · wF · (μ_B − μ_F)²`, which is the textbook form multiplied by `total²`; a constant factor does not move the maximum, so the argmax is the same. Pixels at or above the chosen `t` become 255, the rest become 0.

### Morphological operations

With structuring element `S` (here `np.ones((3,3))`) and set notation for the white pixels:

| Operation | Definition | Function |
|---|---|---|
| Erosion | `A ⊖ S = { z : S_z ⊆ A }` | a pixel survives only if all 9 neighbors are foreground |
| Dilation | `A ⊕ S = { z : S_z ∩ A ≠ ∅ }` | a pixel turns on if any of the 9 neighbors is foreground |
| Opening | `(A ⊖ S) ⊕ S` | erosion then dilation; removes small specks |
| Closing | `(A ⊕ S) ⊖ S` | dilation then erosion; fills small holes |

## Pipeline

One function, `eksperiment()`, applies the same sequence to every image:

1. Grayscale (`to_gray`)
2. Gaussian blur, 5x5, σ = 1 (`gaussian_blur`) to suppress noise before thresholding
3. Otsu threshold on the blurred image (`otsu_segment`), which also prints the threshold it found
4. Convert to 0/1 and run erosion, dilation, opening, closing
5. Display: original against the raw Otsu mask, then the mask against the four morphological results

Blur before threshold is the correct order. Without it, single noisy pixels pull the histogram around and Otsu places the cut in the wrong place; the manual convolution is what makes that step possible here.

## Results

| Image | Content | Otsu threshold | Foreground after Otsu | Erosion | Dilation | Opening | Closing |
|---|---|---|---|---|---|---|---|
| bluefire.jpg | flame, dark sky | 101 | 15.1% | 12.9% | 17.3% | 14.9% | 15.1% |
| handwrittenimage.jpeg | text, white paper | 181 | 88.7% | 79.6% | 95.4% | 87.5% | 89.0% |
| jakartamacet .webp | traffic scene | 108 | 48.5% | 38.0% | 58.8% | 46.9% | 49.2% |
| pov top sea ship.jpg | tanker from above | 88 | 12.9% | 9.7% | 16.1% | 12.2% | 13.1% |
| satelite view .avif | satellite photo | 94 | 24.9% | 15.9% | 34.7% | 23.5% | 25.7% |
| shapeimage.png | flat shapes, palette PNG | 4 (should be 139, see finding 1) | 9.0% | 4.9% | 13.1% | 8.8% | 9.0% |

Each percentage is the share of white pixels in a 512x512 result, measured by re-running the notebook code on the six files in `images/`. The thresholds are the ones the notebook printed, and a `np.bincount` version of the same search returns the same six values, so the histogram call is not what goes wrong. `shapeimage.png` is a palette PNG, so `to_gray()` handed the pipeline palette indices and Otsu cut at 4; converting to RGB first gives 139 and pushes the foreground share to 49.8%, with the four morphology figures moving with it (erosion 46.7%, dilation 52.1%, opening 49.4%, closing 49.0%).

Across the five photographic images, dilation pushed the foreground boundary outward and erosion pulled it inward, which is the direction each operator is defined to move. Opening came out between erosion and the original, and closing came out slightly above the original. Both hold for all five. The sixth image does not belong in the same sentence: its morphology results describe a mask built from palette indices, not from brightness.

## How to Run

This is a Google Colab notebook. To run it:

1. Open [Google Colab](https://colab.research.google.com/).
2. Click **File** > **Upload notebook**.
3. Upload `PCD_Assignment03.ipynb`.
4. Run all cells (**Runtime** > **Run all**).

The notebook downloads the six images from Google Drive at the end, resizes each to 512x512, and runs the pipeline on it.

> **Note:** No local install is needed. `numpy`, `Pillow`, and `matplotlib` are already in the Colab image. The cells do not import OpenCV; all operators are written by hand.

> **Note:** Images are fetched by Drive file ID through `requests`, and `response.raise_for_status()` will stop the run if a link stops being public. `images/` holds local copies of the same six files, so `resize()` can be pointed at those instead.

## Repository Structure

```
PCD_Assignment3/
├── PCD_Assignment03.ipynb      # Google Colab notebook (main code)
├── images/
│   ├── bluefire.jpg            # 894x894
│   ├── handwrittenimage.jpeg   # 694x442
│   ├── jakartamacet .webp      # 1220x784
│   ├── pov top sea ship.jpg    # 612x456
│   ├── satelite view .avif     # 500x375
│   └── shapeimage.png          # 447x447, palette (33 colors)
└── README.md                   # this file
```

The local files are the exact files behind the six Drive IDs, matched by byte size.

## Dependencies

- Python 3
- NumPy
- Pillow (`PIL`)
- Matplotlib
- Requests
- IPython (`display`)

## Findings and Known Limitations

The notebook runs end to end and produces the results above. Its defects were found by re-running the pipeline locally on the six files and comparing against a reference implementation. They are listed here rather than patched into the notebook, so the submitted version stays the one that was graded.

1. `to_gray()` returns early when `np.array(img)` comes back 2D. That is right for a real grayscale image and wrong for a palette one, which is what `shapeimage.png` is: 33 colors, so the values it hands to the rest of the pipeline are indices between 0 and 32 rather than brightness. Otsu then returns 4, the value printed in the notebook, and only 9.0% of pixels pass `gray >= 4`. Converting to RGB first gives 139 and 49.8% foreground. This is the one output in the notebook that is visibly wrong.
2. Both morphology loops run `range(1, rows-1)` over a canvas of zeros, so the outer 1-pixel frame is black in all four results no matter what the mask looked like there. The interior is correct: against `scipy.ndimage`, the pixel difference inside the frame is 0. But dilation has to contain its input set, and it does not, because the border pixels are discarded: 1925 white pixels lost on `handwrittenimage.jpeg`, 814 on `jakartamacet .webp`, 408 on `pov top sea ship.jpg`, 216 on `satelite view .avif`, 83 on `shapeimage.png`, 31 on `bluefire.jpg`. Erosion staying inside the input set is fine, and it does (0 violations). Closing has the opposite requirement, that it contain the input mask, and it breaks the same way: 3833 pixels of the original mask are missing from the closing result on `handwrittenimage.jpeg`, 2034 on `shapeimage.png`, 1619 on `jakartamacet .webp`, 818 on `pov top sea ship.jpg`, 437 on `satelite view .avif`, 62 on `bluefire.jpg`. Opening holds the other direction (0 violations).
3. `binary[gray >= T] = 255` labels the bright class as the object. That works for a flame on a dark sky, where 15.1% comes out white. For handwriting on white paper it is backwards: 88.7% of the frame is labelled foreground, so the "object" is the paper. Erosion and dilation then act on the paper around the ink (79.6% and 95.4%), and nothing in the output or the code says so. The threshold itself, 181, is fine; the labeling convention and the missing inversion check are the problem.
4. The only number the notebook prints is the threshold. Assignment 02 evaluated its results with four metrics, so claims like "opening removes specks" or "closing fills holes" would have been checkable there and are not checkable here. Foreground ratio, connected-component count, and boundary length before and after each operation would fix that.
5. Every image is forced to 512x512. `jakartamacet .webp` comes in at 1220x784 and loses 36% of its width, `handwrittenimage.jpeg` at 694x442 loses the same, `satelite view .avif` at 500x375 loses 25%. Only `bluefire.jpg` is square to begin with.
6. `convolve()` pads with `mode='edge'` and returns output for the full frame, while morphology skips the frame entirely. Two conventions inside one pipeline.
7. `output.astype(np.uint8)` floors every value instead of rounding it, which darkens the blurred image by 0.34 to 0.50 intensity levels on average across the six images. No threshold changed because of it, but `np.rint()` before the cast costs nothing.
8. The loops are pure Python. Blur plus Otsu takes 0.4 s per image and the four morphological operations take 2.3 to 2.9 s, about 20.7 s for the whole notebook body. At 512x512 that is tolerable; a slicing-based or `cv2.morphologyEx` version would run roughly a hundred times faster.

One thing that looks suspicious but is harmless: `np.histogram(..., bins=256, range=(0, 255))` spreads 256 bins over 254 levels instead of 256, so bin width is 254/256 and the bin index is not exactly the intensity. The inputs are integers, so the index still lands on the same value. A `np.bincount` version of Otsu returns the same threshold on all six images, including the palette one: 101, 181, 108, 88, 94, and 4.

## Recommendations

- Convert to RGB (or L) before array conversion: `Image.open(...).convert("RGB")`. A 2D array should only be trusted as brightness when `img.mode == "L"`.
- Pad the binary with `np.pad(binary, 1)` and loop over the full frame, or use `scipy.ndimage` with `border_value=0`. Either way dilation can then grow into the border.
- Decide polarity explicitly. If the foreground fraction exceeds 0.5, invert the mask before morphology, and print which branch was taken.
- Keep the aspect ratio when resizing (`.thumbnail()` or letterboxing), or drop the resize and report the original size.
- Add three metrics per operation: white-pixel ratio, connected-component count, and perimeter. That is what makes opening and closing arguments checkable.
- Vectorize morphology with slicing or `cv2.morphologyEx`, and blur with `cv2.filter2D`.
- Round before casting to `uint8`.

## Conclusion

The five operators and the convolution kernel work, and the interior of every result matches a reference implementation. The pipeline order is right: blur first, threshold second, morphology last.

What fails is everything around the operators. A palette PNG slips past the grayscale check and reports a threshold of 4. A 1-pixel frame is silently excluded from both morphological loops, which is enough to make dilation lose pixels it should keep. One of the six images is segmented inside out because the polarity is hardcoded to the bright class. And with no metrics, none of this shows up in the notebook's own output; it only appears when the code is run again and the numbers are compared.

Writing the operators was the easy half of this assignment. The other half is checking what they produce, and that is where the notebook is thin.