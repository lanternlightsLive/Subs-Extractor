import argparse
import csv
import re
from collections import defaultdict
from pathlib import Path

STAMP = r'\[(\d{4}-\d\d-\d\d \d\d:\d\d:\d\d)\]'
GIFT = re.compile(STAMP + r' \[Notification\] (\w+) gifted a Tier (\d+) sub to (\w+)!', re.I)
BATCH = re.compile(STAMP + r' \[Usernotice\] (\w+) is gifting (\d+) Tier (\d+) Subs? to ', re.I)
SUB = re.compile(STAMP + r' \[Notification\] (\w+) subscribed at Tier (\d+)\.', re.I)
MONTHS = re.compile(r"They've subscribed for (\d+) months", re.I)

def extract(path):
    rows, batches = [], []
    for line in path.read_text(encoding='utf-8-sig', errors='replace').splitlines():
        if m := GIFT.search(line):
            date, giver, tier, recipient = m.groups()
            rows.append(dict(timestamp=date, username=giver, event='Gifted Sub', quantity=1, tier=tier, recipient=recipient, months=''))
        elif m := BATCH.search(line):
            date, giver, quantity, tier = m.groups()
            batches.append(dict(timestamp=date, username=giver, quantity=int(quantity), tier=tier))
        elif m := SUB.search(line):
            date, username, tier = m.groups()
            months = MONTHS.search(line)
            rows.append(dict(timestamp=date, username=username, event='Resubscription' if months and int(months.group(1)) > 1 else 'Subscription', quantity=1, tier=tier, recipient='', months=months.group(1) if months else ''))
    # Batch announcements are metadata, not additional subs. If the log is
    # incomplete, preserve unmatched gift counts as a separate estimated row.
    for batch in batches:
        matching = sum(1 for r in rows if r['event'] == 'Gifted Sub' and r['username'].lower() == batch['username'].lower() and r['tier'] == batch['tier'] and abs((__import__('datetime').datetime.fromisoformat(r['timestamp']) - __import__('datetime').datetime.fromisoformat(batch['timestamp'])).total_seconds()) <= 30)
        if matching < batch['quantity']:
            rows.append(dict(timestamp=batch['timestamp'], username=batch['username'], event='Gifted Sub (recipients missing)', quantity=batch['quantity'] - matching, tier=batch['tier'], recipient='', months=''))
    rows.sort(key=lambda r:r['timestamp'])
    return rows

def main():
    parser = argparse.ArgumentParser(description='Extract Twitch subscription events from chat logs')
    parser.add_argument('files', nargs='+', type=Path)
    parser.add_argument('--out', default='subscription_events.csv')
    args = parser.parse_args()
    rows = []
    for file in args.files:
        rows.extend(extract(file))
    fields = ['timestamp','username','event','quantity','tier','recipient','months']
    with open(args.out, 'w', encoding='utf-8-sig', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=fields)
        writer.writeheader()
        writer.writerows(rows)
    totals = defaultdict(lambda: {'subscriptions':0,'resubscriptions':0,'gifted_subs':0})
    for row in rows:
        col = 'gifted_subs' if row['event'].startswith('Gifted') else 'resubscriptions' if row['event']=='Resubscription' else 'subscriptions'
        totals[row['username']][col] += row['quantity']
    summary_path = Path(args.out).with_name('subscription_summary.csv')
    with summary_path.open('w', encoding='utf-8-sig', newline='') as f:
        writer = csv.writer(f)
        writer.writerow(['username','subscriptions','resubscriptions','gifted_subs','total'])
        for name, counts in sorted(totals.items(), key=lambda item:item[0].lower()):
            writer.writerow([name, *counts.values(), sum(counts.values())])
    print(f'Extracted {len(rows)} events; {sum(int(r["quantity"]) for r in rows)} subscriptions total')
    print(f'Saved {args.out} and {summary_path}')

if __name__ == '__main__':
    main()
