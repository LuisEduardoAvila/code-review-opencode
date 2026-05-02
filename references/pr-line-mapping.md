# PR Line Mapping

Map findings from merged file lines to GitHub diff positions.

## The Problem

GitHub PR review comments use **diff positions**, not file line numbers. You must convert findings from "line 42 in the merged file" to "position 5 in the diff".

### Why Not Line Numbers?

The diff position system allows GitHub to:
- Handle moved code blocks
- Track comments across file renames
- Support multi-line comments
- Work with partial diffs (only showing changed regions)

## Diff Position System

### Position Counter

For each file in a diff:

```
@@ -10,5 +10,7 @@ function example() {
     const x = 1;           <- position 1
     const y = 2;           <- position 2
+    const z = 3;           <- position 3 (added)
+    const w = 4;           <- position 4 (added)
     return x + y;          <- position 5
-    console.log(x);        <- position 6 (removed)
+    console.log(y);        <- position 7 (added)
 }
```

**Rules:**
1. Position starts at 1 after each `@@` header
2. Count ALL lines (added, removed, unchanged context)
3. Blank lines in the diff count toward position
4. Position resets at each new file (`diff --git` line)
5. Position resets at each new hunk (`@@` line)

### Position vs Line Number

| Concept | Example | Use Case |
|---------|----------|-----------|
| **File line number** | 42 | "This is line 42 in the actual file" |
| **Diff position** | 5 | "This is the 5th line in this diff hunk" |
| **Old line number** | 10 | "Before changes, this was line 10" |
| **New line number** | 12 | "After changes, this is line 12" |

## Mapping Algorithm

### Input

- `diff_content`: Raw diff string for the file
- `file_path`: Target file path
- `merged_line`: Line number in the merged file (1-indexed)

### Output

- `position`: Diff position (1-indexed) if in changed region
- `None`: If line is not in diff (unchanged outside hunks)

### Algorithm Steps

```python
def map_line_to_position(diff_content, merged_line):
    """
    Map a merged file line number to diff position.
    
    Returns: (position, line_type) or (None, None) if not in diff
    """
    position = 0
    current_line = 0
    
    for line in diff_content.split('\n'):
        # Skip file headers
        if line.startswith('diff --git'):
            position = 0
            current_line = 0
            continue
        
        if line.startswith('---') or line.startswith('+++'):
            continue
        
        # Parse hunk header: @@ -a,b +c,d @@
        if line.startswith('@@'):
            match = re.match(r'@@ -\d+,?\d* \+(\d+),?\d* @@', line)
            if match:
                current_line = int(match.group(1)) - 1  # Will be incremented at first line
                position = 0  # Reset for new hunk
                continue
        
        # Skip non-content lines
        if not line:
            continue
        
        # Increment position for all diff lines
        position += 1
        
        # Track which file line we're at
        line_type = None
        if line.startswith('+'):
            line_type = 'added'
            # Added lines advance the new file line counter
            current_line += 1
        elif line.startswith('-'):
            line_type = 'removed'
            # Removed lines don't advance new file counter
            # (they existed in old file, not in new)
        else:
            line_type = 'context'
            current_line += 1
        
        # Check if this is our target line
        if current_line == merged_line:
            if line_type in ('added', 'context'):
                return (position, line_type)
            else:
                # Removed line - not in merged file
                return (None, None)
    
    return (None, None)
```

### Complete Python Implementation

```python
import re
from typing import Optional, Tuple

def build_line_map(diff_content: str) -> dict:
    """
    Build a mapping of merged line numbers to diff positions.
    
    Returns dict: {line_number: (position, line_type)}
    """
    line_map = {}
    position = 0
    current_line = 0
    
    for line in diff_content.split('\n'):
        # New file
        if line.startswith('diff --git'):
            position = 0
            current_line = 0
            continue
        
        # Skip headers
        if line.startswith('---') or line.startswith('+++'):
            continue
        
        # New hunk
        if line.startswith('@@'):
            match = re.match(r'@@ -\d+,?\d* \+(\d+),?\d* @@', line)
            if match:
                current_line = int(match.group(1))
                position = 0
            continue
        
        # Skip empty diff metadata
        if not line or line.startswith('index ') or line.startswith('new file'):
            continue
        
        # Count position
        position += 1
        
        # Determine line type
        if line.startswith('+'):
            line_map[current_line] = (position, 'added')
            current_line += 1
        elif line.startswith('-'):
            # Removed lines don't appear in merged file
            pass
        else:
            # Context line (starts with space or nothing)
            line_map[current_line] = (position, 'context')
            current_line += 1
    
    return line_map


def find_diff_position(line_map: dict, merged_line: int) -> Optional[int]:
    """
    Look up diff position for a merged line number.
    
    Returns position or None if not in diff.
    """
    if merged_line in line_map:
        position, line_type = line_map[merged_line]
        return position
    return None
```

## Usage Examples

### Example 1: Simple Addition

**Diff:**
```
@@ -10,3 +10,5 @@ function add(a, b) {
     return a + b;
+    console.log(a, b);  // added
+    return result;       // added
 }
```

**Line map:**
```
11 -> (1, 'context')  // return a + b;
12 -> (2, 'added')    // console.log
13 -> (3, 'added')    // return result
```

**Query:** "Line 12 in merged file"
**Result:** Position 2

### Example 2: Modification

**Diff:**
```
@@ -5,4 +5,4 @@ const config = {
-    timeout: 5000
+    timeout: 10000
 };
```

**Line map:**
```
6 -> (1, 'context')  // };
7 -> (2, 'added')    // timeout: 10000
```

**Query:** "Line 6 in merged file" (the `timeout` line)
**Result:** Position 2

### Example 3: Multiple Hunks

**Diff:**
```
@@ -10,3 +10,4 @@ function init() {
     setup();
+    validate();  // added
 }
@@ -50,2 +51,3 @@ function process() {
     handleResult();
+    logResult();  // added
 }
```

**Line maps:**
```
First hunk:
11 -> (1, 'context')
12 -> (2, 'added')
13 -> (3, 'context')

Second hunk:
52 -> (1, 'context')
53 -> (2, 'added')
```

## Edge Cases

### Case: Line Not in Diff

**Finding:** "Line 100 in merged file"
**Diff:** Only shows lines 10-20 and 50-60
**Action:** Return `None`, add to summary body only

```python
if position is None:
    # Line is outside diff context
    # Include in summary body, not as inline comment
    pass
```

### Case: Removed Line

**Diff:**
```
-    oldFunction();
+    newFunction();
```

**Finding:** References the removed line
**Action:** Skip or reference in summary

```python
# Removed lines don't appear in merged file
# If finding references old code being deleted, it's not applicable
```

### Case: Multi-Line Comment

GitHub supports multi-line comments (start_line, line, start_side, side):

```json
{
  "path": "file.ts",
  "start_line": 10,
  "line": 15,
  "start_side": "RIGHT",
  "side": "RIGHT",
  "body": "This entire block has issues"
}
```

Map both start and end lines separately.

### Case: File Renamed

**Diff:**
```
diff --git a/old.ts b/new.ts
--- a/old.ts
+++ b/new.ts
```

**Action:** Use `new.ts` path for comments, track that file was renamed.

## Integration with Review Workflow

### Step 1: Build Maps Per File

```python
file_maps = {}
for file_path, diff in diffs.items():
    file_maps[file_path] = build_line_map(diff)
```

### Step 2: Translate Findings

```python
for finding in findings:
    path = finding['location'].split(':')[0]
    line = int(finding['location'].split(':')[1])
    
    position = find_diff_position(file_maps[path], line)
    
    if position:
        comments.append({
            "path": path,
            "position": position,
            "body": format_comment(finding)
        })
    else:
        # Add to summary body
        summary_findings.append(finding)
```

### Step 3: Format Comments

```python
def format_comment(finding):
    severity = finding['severity']
    
    if severity == 'P0':
        prefix = "🚨 **Critical**"
    elif severity == 'P1':
        prefix = "⚠️ **Important**"
    else:
        prefix = "💡 **Suggestion**"
    
    return f"{prefix}: {finding['issue']}\n\n{finding['suggestion']}"
```

## Performance

For large PRs, precompute all line maps once:

```python
# Do this once before running roles
for file_path in changed_files:
    diff = get_diff(file_path)
    line_maps[file_path] = build_line_map(diff)

# Then lookup is O(1)
position = find_diff_position(line_maps[path], line)
```

## Testing

### Test Case 1

```python
diff = """@@ -1,3 +1,4 @@
 one
-two
+two
+three
"""

line_map = build_line_map(diff)
# Expected: {1: (1, 'context'), 2: (2, 'added'), 3: (3, 'added')}

assert find_diff_position(line_map, 2) == 2
assert find_diff_position(line_map, 99) is None
```

### Test Case 2

```python
diff = """diff --git a/test.ts b/test.ts
@@ -10,2 +10,3 @@
+added line
 context"""

line_map = build_line_map(diff)
assert find_diff_position(line_map, 10) == 1  # added line
assert find_diff_position(line_map, 11) == 2  # context
```