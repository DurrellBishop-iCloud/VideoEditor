# Three Line Edit - Development Notes

## Current Version: v2.9.8

## Project Overview
A browser-based video editor with a three-stripe timeline interface. Videos are loaded, frames are extracted, and users can delete/move sections using mark in/out points.

## Architecture

### Frame Storage
- `state.frames[]` - Array of ALL frames from the original video
- Each frame object:
  ```javascript
  {
    time: 1.533,        // Actual video time this frame was captured at
    image: Image,       // Frame image (80px initially, 240px after HD swap)
    deleted: false,     // true when user deletes
    effectiveTime: ...  // Time in edited timeline
  }
  ```

### Time Systems
1. **Source Time** - Original video time (0 to duration)
2. **Effective Time** - Edited timeline time (accounts for deletions)
3. **Frame Time** - The actual `frame.time` when a frame was captured

### Key Functions
| Function | Location | Purpose |
|----------|----------|---------|
| `extractFrames()` | ~line 886 | Play-through thumbnail extraction (80px) |
| `extractHDFramesInBackground()` | ~line 1082 | Background HD extraction (240px) |
| `swapToHDFrames()` | ~line 1261 | Atomic swap to HD frames |
| `getFrameAtTime(time)` | ~line 1318 | Find closest non-deleted frame |
| `effectiveToSource(time)` | ~line 765 | Convert effective → source time |
| `sourceToEffective(time)` | ~line 785 | Convert source → effective time |
| `getSelection()` | ~line 1819 | Get current in/out times for operations |
| `drawPreview()` | ~line 1050 | Draw main preview frame |
| `drawStripe()` | ~line 1356 | Draw timeline stripes |

## Recent Fixes (This Session)

### v2.9.8 - Edit List Playback Preview
- Tap blue edit list stripe to play preview of edited sequence
- Plays through all clips in order at real-time speed
- Green playback position indicator on stripe
- Time display shows edit list progress
- Touch bottom stripes anytime to stop and return to normal editing

### v2.9.7 - HD Frame Background Extraction
- **Complete rewrite of HD frame loading**
- Thumbnails (80px) extracted first via play-through - user can start editing immediately
- HD frames (240px) extracted in background using cloned video element
- When HD extraction completes, atomic swap replaces all frames at once
- Edit states (deleted, effectiveTime) copied from old frames to HD frames
- Progress indicator shows "HD: 45%" during extraction
- Removed old seeking-based HD loader (was disabled due to sync issues)
- New state variables: `hdFrames`, `hdExtractionProgress`, `hdExtractionComplete`
- New functions: `extractHDFramesInBackground()`, `loadHDFrameImages()`, `swapToHDFrames()`

### v2.9.6 - Marker Frame Times
- Mark In/Out now use actual displayed frame's `frame.time`
- Delete/Move use `<=` for inclusive range (both in and out frames deleted)

### v2.9.5 - Delete Frame Accuracy
- `getSelection()` finds actual displayed frame when no markers set
- Ensures delete targets exactly the visible frame

### v2.9.4 - First Frame Capture
- Captures frame 1 at time 0 before playback starts
- Fixed off-by-one where first frame was missed

### v2.9.3 - Frame Info Display
- Shows "Frame: 123 | Time: 4.100s" in top-right corner
- Helps verify frame alignment during testing

### v2.9.2 - Play-Through Extraction
- **BREAKING CHANGE**: Frame extraction now plays video in real-time
- Uses `requestVideoFrameCallback` for frame-accurate capture
- Falls back to `timeupdate` for older browsers
- 60s video = 60s extraction time
- Fixes duplicate/missing frames from imprecise browser seeking

### v2.9.1 - Store Actual Time
- Frame extraction stores `video.currentTime` (actual) not requested time

### v2.9.0 - Skip Deleted in getFrameAtTime
- `getFrameAtTime()` now skips deleted frames
- Fixes black preview after deleting

### v2.8.9 - HD Loader Disabled
- `enableHDLoader = false` flag at ~line 1219
- HD frame loading disabled for testing
- Set to `true` to re-enable

### v2.8.8 - Reverted hdImage in Fine Stripe
- Fine stripe uses only `frame.image` (thumbnails)
- HD images were causing sync issues

## Known Issues / Watch Points

### Browser Video Seeking
- Browser `video.currentTime = X` is imprecise
- That's why we use play-through extraction now
- Seeking-based extraction showed frames like "73, 73, 74, 76, 67"

### HD Frame Loading (v2.9.7)
- HD extraction now uses play-through (same as thumbnails) - no seeking issues
- Runs on cloned video element - doesn't interfere with user playback
- Atomic swap after 100% complete - no mixing of HD/non-HD frames
- Edit states (deleted, markers) preserved during swap

### Frame Time vs Calculated Time
- Play-through extraction captures at actual video times
- These don't perfectly align with calculated `frameIndex * frameInterval`
- All operations must use actual `frame.time`, not calculated times
- Fixed in getSelection(), Mark In, Mark Out

## Test Video
`frame_counter_60s.mp4` - 60 second test video at 30fps (1800 frames)
- Each frame displays its sequential number (1-1800)
- Use for testing frame accuracy

## File Structure
```
Three_Line_Edit/
├── index.html          # Main application (single file)
├── DEVELOPMENT.md      # This file
├── frame_counter_60s.mp4  # Test video
└── test_video.mp4      # User's test video
```

## Key Code Sections in index.html

| Line Range | Section |
|------------|---------|
| 1-500 | CSS styles |
| 500-605 | HTML structure |
| 720-745 | State object (includes HD state) |
| 745-800 | Time conversion functions |
| 886-1040 | Frame extraction (play-through thumbnails) |
| 1050-1070 | Preview drawing |
| 1082-1260 | HD background extraction + swap |
| 1318-1355 | getFrameAtTime() |
| 1356-1490 | Stripe drawing |
| 1819-1860 | getSelection(), marker display |
| 1880-1910 | Mark In/Out buttons |
| 1925-1980 | Delete/Move operations |

## Commands Reference

### Git
```bash
cd "/Users/durrellbishop/Documents/Documents - Durrell's MacBook Pro/Projects - Work/videoEditing/Three_Line_Edit"
git status
git add index.html && git commit -m "message" && git push
```

### Create Test Video
```bash
ffmpeg -y -f lavfi -i color=c=black:s=1920x1080:r=30:d=60 \
  -vf "drawtext=fontfile=/System/Library/Fonts/Helvetica.ttc:text='%{frame_num}':start_number=1:x=(w-text_w)/2:y=(h-text_h)/2:fontsize=300:fontcolor=white" \
  -c:v libx264 -pix_fmt yuv420p frame_counter_60s.mp4
```

## Next Steps / TODO

1. ~~**Re-enable HD Loader**~~ - DONE (v2.9.7) - Reimplemented with play-through extraction
2. **Performance** - Play-through extraction is slow (real-time); consider caching
3. **Frame Display** - Remove frame info display when debugging complete
4. **Edit List** - Test MOVE functionality with new frame time fixes
5. **Test HD Swap** - Verify edit states preserved after HD swap

## Session Summary

Main issue solved: Frame extraction was using imprecise browser seeking, causing duplicate and missing frames. Switched to play-through extraction using `requestVideoFrameCallback` which captures every frame accurately.

Secondary issues fixed:
- Delete/Mark operations used calculated times instead of actual frame times
- First frame was being missed (off-by-one)
- Delete wasn't inclusive of out frame
- getFrameAtTime() wasn't skipping deleted frames
