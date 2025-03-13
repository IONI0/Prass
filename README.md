# Prass (Advanced TPP Fork)
Prass with fixes for common Timing Post Processor issues and additional features using more advanced logic. Made for, and primarily tested with Hidive subtitles, but should work for other cases. For manual timing, [Timing Assistant](https://phoscity.github.io/Aegisub-Scripts/Timing%20Assistant/) is recommended. While this should be better for full automation use cases than normal tpp, errors should still be expected.

### Changes
All other prass functionality other than TPP remains unchanged. Main TPP changes include:
- Non-sequential joining logic that is time based.
- Subtitles on bottom and top alignment are processed separately with the option to cross-section-snap them.
- Smarter keyframe snapping to avoid overlap near keyframe.
- Keyframe format v1 support.

### Usage
Only use TPP for dialogue lines and snap simple signs separately unless using `--hidive`.

```console
$ python prass.py tpp --help
Usage: prass.py tpp [OPTIONS] INPUT_FILE

  Timing post-processor. Advanced fork with many fixes. Check github for
  explanations. You have to specify keyframes and timecodes (either as a CFR
  value or a timecodes file) if you want keyframe snapping. All parameters
  default to zero so if you don't want something - just don't put it in the
  command line.

  To add lead-in and lead-out:
  $ prass tpp input.ass --lead-in 50 --lead-out 150 -o output.ass
  To make adjacent lines continuous, with 80% bias to changing end time of the first line:
  $ prass tpp input.ass --overlap 50 --gap 200 --bias 80 -o output.ass
  To snap events to keyframes without a timecodes file:
  $ prass tpp input.ass --keyframes kfs.txt --fps 23.976 --kf-before-end 150 --kf-after-end 150 --kf-before-start 150 --kf-after-start 150 -o output.ass

Options:
  -o, --output <path>
  -s, --style <names>        Style names to process. All by default. Use comma
                             to separate, or supply it multiple times
  --lead-in <ms>             Lead-in value in milliseconds
  --lead-out <ms>            Lead-out value in milliseconds
  --smart-lead-out <ms>      Smart lead-out value in milliseconds. Makes it so
                             that it doesn't add more lead-out if it hits a
                             keyframe
  --overlap <ms>             Maximum overlap for two lines to be made
                             continuous, in milliseconds
  --gap <ms>                 Maximum gap between two lines to be made
                             continuous, in milliseconds
  --bias <percent>           How to set the adjoining of lines. 0 - change
                             start time of the second line, 100 - end time of
                             the first line. Values from 0 to 100 allowed.
                             [0<=x<=100]
  --keyframes <path>         Path to keyframes file
  --timecodes <path>         Path to timecodes file
  --fps <float>              Fps provided as decimal or proper fraction, in
                             case you don't have timecodes
  --kf-before-start <ms>     Max distance between a keyframe and event start
                             for it to be snapped, when keyframe is placed
                             before the event
  --kf-after-start <ms>      Max distance between a keyframe and event start
                             for it to be snapped, when keyframe is placed
                             after the start time
  --kf-before-end <ms>       Max distance between a keyframe and event end for
                             it to be snapped, when keyframe is placed before
                             the end time
  --kf-after-end <ms>        Max distance between a keyframe and event end for
                             it to be snapped, when keyframe is placed after
                             the event
  --cross-section-snap <ms>  How far to snap each top section start or end
                             event to a bottom section event
  --cs-snap-right-cap <ms>   If an an8 or an2 event is unjoined on the end,
                             extend the positive search to this amount
  --hidive                   For working with Hidive subtitles. Runs tpp on
                             styles labeled with Caption in the name
                             seperately and lowers kf_before_end on songs to
                             min(kf_before_end, 250)
  -h, --help                 Show this message and exit.
```

My settings for Hidive scripts:
```console
$ python prass.py tpp "$ass_file" --hidive --lead-in 0 --smart-lead-out 200 --gap 450 --overlap 300 --bias 100 --keyframes "$kf_file" --fps "$fps" --kf-before-start 250 --kf-after-start 250 --kf-before-end 450 --kf-after-end 450 --cross-section-snap 200 --cs-snap-right-cap 350 -o "$out_file"
```

Millisecond values are recommended to be rounded to the nearest 10s.

### Installation
Prass should work on OS X, Linux and Windows without any problems. Prass was originally made for Python 2 but this fork has only been tested for Python 3. The only dependency is [Click](http://click.pocoo.org/3/). Assuming you have python and pip, just run:
```bash
pip install git+https://github.com/IONI0/prass
```

---

### Explanation
Here's an explanation for each of the issues I've fixed so that you can understand why they occur if you've ever experienced them using TPP. Since Prass merely ported Aegisub's implementation, all of these issues should be present there as well. If you want further clarification then don't hesitate to contact me on [discord](https://discord.gg/8v9GBnjdsY).

---

#### Sequential Joining Logic
Vanila TPP's joining logic takes lines chronologically by start time and only joins to the next line in sequence. This is a problem for any instance where lines do not only start after the last one ends. For example, if two events start and end at the same time, only one of them will join to the next line. Here's what it would be like on vanilla tpp:

```
0:00:00.00,0:00:05.00,Line 1      -> 0:00:00.00,0:00:05.00,Line 1
0:00:00.00,0:00:05.00,Line 2      -> 0:00:00.00,0:00:05.20,Line 2
0:00:05.20,0:00:10.00,Join to me! -> 0:00:05.20,0:00:10.00,Join to me!
```

Another example is that when there are multiple speakers, joining becomes very inconsistent.

This fork solves this by making the max_gap and max_overlap joining logic check based on time rather than sequentially.

---

#### Joining Over Keyframe
Since joining is ran before snapping, it is possible for a line near a keyframe to be joined right to another line and escape the kf_snap distance. This is usually not ideal since you would generally prefer for a line to be snapped if it is within distance.

This fork solves this by checking if a line will be snapped to a keyframe and doesn't join if it will.

---

#### Bias Shenanigans
Because joining is done before snapping, a line can get joined and then with the applied bias it will adjust the start time of the next line, this is intended. But the problem is that if the end of the line that just joined then gets snapped back but the start of the other line doesn't, bias has then been applied to the other line even though it isn't joined in the end.

```
Keyframe at 0:00:05.00
kf_before_end = 300ms
kf_before_start = 200ms
bias = 75

0:00:00.00,0:00:05.00,Line 1 -> 0:00:00.00,0:00:05.30,Line 1 -> 0:00:00.00,0:00:05.00,Line 1
0:00:05.40,0:00:10.00,Line 2 -> 0:00:05.30,0:00:10.00,Line 2 -> 0:00:05.30,0:00:10.00,Line 2
```

Another thing is that the bias applied could also bring a line that would otherwise not have snapped into snapping distance, making it snap from farther than you'd like.

This fork solves this using the same fix as the last issue.

---

#### Multiple Sections
Lines that are on the bottom and top track using alignment 2 and 8 are usually different speakers or has a song. Vanilla TPP gets very confused if they are both ran at the same time due to joining being done sequentially. By running them through tpp separately, and with the new joining logic, this allows them to not interfere with the other track. A cross-section-snap can then be applied to simulate how a timer might want lines start and end at the same time if they are close enough to reduce visual burden.

Since the bottom track is usually the main track. `cross-section-snap` works by snapping the top track start or end time to the bottom track by the ms set. Additionally, `cs-snap-right-cap` can be used to extend the snapping range of the end times for both bottom and top events farther. Bottom and top tracks are identified by either the style definition or `\an2 \an8` tags.

---

#### Overlap Near Keyframe
Keyframe snapping can create overlaps if two lines start and end near a keyframe. Normally if the two lines are close enough to the keyframe then they would both snap. If the two lines are far enough from the keyframe then they would both not snap. But if they are at a middle distance away where the settings only allow one of the them to snap, then this creates the overlap.

The cause is when values `kf_after_end > kf_after_start` or `kf_before_start > kf_before_end`. Most timing guides advocate for higher `kf_after_end` than `kf_after_start` so this can be a big problem. If you are using vanilla TPP then be vigilant about this issue or change your snap settings to where they would not cause the issue.

```
Keyframe at 0:00:05.00
kf_after_end = 450ms
kf_after_start = 200ms

0:00:00.00,0:00:04.60,Line 1 -> 0:00:00.00,0:00:05.00,Line 1
0:00:04.60,0:00:10.00,Line 2 -> 0:00:04.60,0:00:10.00,Line 2
```

```
Keyframe at 0:00:05.00
kf_before_start = 450ms
kf_before_end = 200ms

0:00:00.00,0:00:05.40,Line 1 -> 0:00:00.00,0:00:05.40,Line 1
0:00:05.40,0:00:10.00,Line 2 -> 0:00:05.00,0:00:10.00,Line 2
```

This fork solves this by detecting if line that is about to snap is in this middle distance away and whether it is joined to another line that won't snap. It then skips snapping if it would have created an overlap.

---

#### Keyframe Format v1 Support
Support for keyframes is in this form:
```
# keyframe format v1
fps 0
0
72
113
229
312
```