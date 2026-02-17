#!/bin/bash

# MiniMax Usage - Check coding plan credits remaining
# Usage: minimax-usage [--threshold <percent>]
#   --threshold: Only output when remaining % drops below this value
#                If omitted, always outputs (backward compat)

API_KEY="${MINIMAX_API_KEY}"
THRESHOLD=""

# Parse args
while [[ $# -gt 0 ]]; do
  case "$1" in
    --threshold)
      THRESHOLD="$2"
      shift 2
      ;;
    *)
      echo "Unknown option: $1" >&2
      exit 1
      ;;
  esac
done

if [[ -z "$API_KEY" ]]; then
  echo "Error: MINIMAX_API_KEY environment variable not set" >&2
  exit 1
fi

response=$(curl -s --location 'https://www.minimax.io/v1/api/openplatform/coding_plan/remains' \
  --header "Authorization: Bearer $API_KEY" \
  --header 'Content-Type: application/json')

if [[ $? -ne 0 ]] || [[ -z "$response" ]]; then
  echo "Error: Failed to reach MiniMax API" >&2
  exit 1
fi

if [[ "$response" == *"error"* ]] || [[ "$response" == *"Error"* ]]; then
  echo "API Error: $response" >&2
  exit 1
fi

echo "$response" | python3 -c "
import sys
import json
from datetime import datetime, timezone, timedelta

data = json.load(sys.stdin)
et = timezone(timedelta(hours=-5))
threshold = ${THRESHOLD:-0}
has_threshold = bool('${THRESHOLD}')

if 'model_remains' in data and len(data['model_remains']) > 0:
    mr = data['model_remains'][0]
    total = mr.get('current_interval_total_count', 0)
    remaining = mr.get('current_interval_usage_count', 0)  # API returns remaining, not used
    used = total - remaining
    model = mr.get('model_name', 'MiniMax')

    remaining_pct = (remaining / total * 100) if total > 0 else 0

    # If threshold set and we're above it, exit silently
    if has_threshold and remaining_pct >= threshold:
        sys.exit(0)

    lines = []
    if has_threshold and remaining_pct < threshold:
        lines.append(f'**⚠️ MiniMax Usage Alert — {model}**')
        lines.append(f'Remaining: **{remaining:,}** of {total:,} requests ({remaining_pct:.1f}%)')
    else:
        lines.append(f'**🤖 MiniMax Usage — {model}**')
        lines.append(f'Remaining: **{remaining:,}** of {total:,} requests ({remaining_pct:.1f}%)')

    if 'end_time' in mr:
        end_dt = datetime.fromtimestamp(mr['end_time'] / 1000, tz=et)
        lines.append(f'Resets: {end_dt.strftime(\"%b %d, %Y %I:%M %p\")} ET')

    if 'remains_time' in mr:
        secs = int(mr['remains_time'] / 1000)
        h, m, s = secs // 3600, (secs % 3600) // 60, secs % 60
        lines.append(f'Time left: {h}:{m:02d}:{s:02d}')

    print('\n'.join(lines))
else:
    print(json.dumps(data, indent=2))
"
