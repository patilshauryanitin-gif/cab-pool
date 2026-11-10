# Cab Pool Matcher (SUTT Back-End Task 1)

A command-line app in Python that helps students share cabs at the end of the semester. It tracks the available cabs, collects ride requests, and fills cabs with students going to the same place at the same time. When seats are short, **seniors get priority**.

## Requirements

- Python 3.7 or newer
- No external packages (standard library only: `heapq`, `json`, `dataclasses`, `pathlib`, `datetime`)

## How to run

```bash
git clone [your-repo-url]
cd [your-repo-folder]
python main.py
```

On startup the app loads the cabs from `cabs.json`, then restores the previous session from `saved_state.json` if it exists.

## Menu

| Option | What it does |
|---|---|
| 1. Add a ride request | Asks for name, destination, time slot and year, then adds the student to that trip's waiting queue |
| 2. View waiting students and all cabs | Lists everyone waiting (most senior first) and every cab, open or left, with seats taken like `3/4` |
| 3. Find matches (preview only) | Shows who would get a seat in each open cab right now. Nobody is assigned and the queues do not change |
| 4. Dispatch a cab for a destination and time | Picks the first open cab for that trip, fills it with the most senior waiting students, and sends it off |
| 5. Dispatch a specific open cab | Same as option 4, but you choose which open cab (useful when a trip has several) |
| 6. Quit | Saves and exits |

## Valid inputs

- **Destination:** Delhi, Jaipur
- **Time slot:** Morning, Afternoon, Evening, Night
- **Year:** a whole number from 1 to 5
- **Name:** cannot be empty

Anything else shows a friendly "Input error" message and returns to the menu. The app does not crash on bad input.

## Cab file format (`cabs.json`)

A JSON list of cabs, loaded in file order:

```json
[
  {"cab_id": "C1", "driver": "Ramesh", "seats": 4, "destination": "Delhi",  "time_slot": "Morning"},
  {"cab_id": "C2", "driver": "Suresh", "seats": 3, "destination": "Delhi",  "time_slot": "Morning"},
  {"cab_id": "C3", "driver": "Mahesh", "seats": 4, "destination": "Jaipur", "time_slot": "Night"}
]
```

- `seats` must be 1 to 4. Cabs can have different seat counts.
- `destination` and `time_slot` must be one of the valid values above.
- If the file is missing, empty or invalid, the app prints the reason and starts with no cabs instead of crashing.

## How the program works

### Classes

- **`RideRequest`**: name, destination, time slot, year (1 to 5), submission time, and a `sequence` number that increases with every request.
- **`Cab`**: cab ID, driver, seats, destination, time slot, a `has_left` flag, and a plain list of passengers.
- **`CabPoolMatcher`**: holds the cabs and the queues, and contains all the matching logic.

### Requests wait in a queue first

A new request does not get a cab immediately. It joins the waiting queue for its trip. Cabs are filled only when a cab is dispatched, so a senior who signs up late can still get a seat ahead of a junior who signed up earlier.

### One priority queue per trip

Each (destination, time slot) pair, like `("Delhi", "Morning")`, has its own **max-heap**, stored in a dictionary (`self.queues`). The queue belongs to the trip, not to any one cab.

Python's `heapq` is a min-heap, so each entry is stored as:

```
(-year, sequence, request)
```

- `-year` makes the most senior student come out first (year 5 gives -5, the smallest value).
- `sequence` is a unique counter. It breaks ties between students of the same year, so whoever asked first gets the seat first.
- Because `sequence` is never repeated, Python never has to compare two `RideRequest` objects, which would crash.

### Find matches (preview)

For each open cab, in the same order as `cabs.json`, the app takes the most senior waiting students for that cab's trip, up to its number of seats. If several open cabs serve the same trip, each one shows the next students in line (no student is shown twice). This matches exactly who dispatching those cabs one after another would take. The preview never removes anyone from a queue.

### Dispatch a cab

1. The user picks a destination and time slot (and, in option 5, a specific open cab).
2. The app finds the first matching open cab, in file order.
3. It pops up to `seats` students from that trip's heap, most senior first.
4. They are added to the cab's passenger list and the cab is marked as left.

If no open cab exists for the trip, or nobody is waiting, the app prints a clear message and changes nothing. Once a cab has left it is never offered again, so nobody new can be added to it.

### Saving and loading

State is saved to `saved_state.json` after every change (adding a request, dispatching a cab) and when quitting, including with Ctrl+C or Ctrl+D. It stores:

- who is still waiting, with their original `sequence` numbers,
- which students are in each cab,
- which cabs have left,
- the next `sequence` number.

On the next start the state is restored, so the user continues exactly where they left off. Because each waiting student keeps their original `sequence` number, place in line survives a restart: a 3rd year who asked before another 3rd year is still ahead of them.

The file is written to a temporary file first and then renamed over the old one, so a crash in the middle of a save cannot leave a half-written file.

If the save file is missing or damaged, the app does not crash. It tells the user and starts a fresh session.

**Why JSON?** It is human-readable (you can open it and check it), it maps directly onto Python dicts and lists, and it is safe to load. `pickle` is unreadable and can run arbitrary code when loaded, so I avoided it.

## Tricky situations

**1. Half-empty cab.** Any open cab can be dispatched even if it will not be full. It takes everyone waiting for its trip, up to its seats, and leaves with fewer passengers. The unused seats are simply empty, and the cab list shows something like `2/4 seats | Left`.

**2. No cabs left.** If every cab for a trip has already left, new requests still join that trip's queue and are never dropped. The app tells the user that there is no open cab for the trip right now and that they remain on the waiting list. They stay visible in option 2, and they would be seated as soon as an open cab for that trip exists.

**3. Same year.** Every request gets an increasing `sequence` number, used as the second item of the heap entry. Among students of the same year, the smaller `sequence` comes out first, so the one who asked first gets the seat. The number is saved to disk, so this also holds after a restart.

**4. Lots of students (1,000 waiting).** Let `n` be the number of requests waiting for one trip, and `C` the number of cabs (a small number).

| Operation | Time | Why |
|---|---|---|
| Adding one request | O(log n) | One `heappush` |
| Adding all 1,000 | O(n log n) | 1,000 pushes, each O(log n) |
| Dispatching one cab (up to 4 students) | O(k log n), which is O(log n) | At most k = 4 `heappop` calls, each O(log n). k is a constant because a cab has at most 4 seats |
| Finding an open cab for a trip | O(C) | `open_cabs` scans the cab list once. C is small, so this is fast |

Note: because the app saves after every change, each save also sorts and writes all waiting requests (O(n log n)). So with autosave, adding 1,000 requests one by one costs about O(n² log n) in total. This is still instant for 1,000 students, and it is the price of never losing data on a crash.

## Project structure

```
main.py             the whole app (classes, matching logic, menu)
cabs.json           sample cab data
saved_state.json    saved session (created automatically)
README.md
```

## AI assistance disclosure

This README was written with the help of Claude (an AI assistant made by Anthropic). Claude also reviewed my `main.py` and suggested improvements, which I then applied. I understand how the program works and can explain any part of it.

---

**Author:** [shaurya patil]

