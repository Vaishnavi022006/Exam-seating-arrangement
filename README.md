# AI Based Exam Seating Arrangement

A web application that automatically generates exam seating plans. The seating logic is a **constraint satisfaction problem solved in SWI-Prolog with CLP(FD)**, served through a lightweight **Node.js** backend and an interactive **HTML/CSS/JavaScript** front end.

Students from the same class or taking the same subject are never seated next to each other, which helps reduce copying during exams.

---

## Features

- **Constraint-based seating** using Prolog's `clpfd` library
- **Multiple rooms** with custom rows and columns, and click-to-block seats on an interactive mini grid
- **Roll number / PIN ranges**: enter a starting and ending PIN (for example `21B91A0501` to `21B91A0530`) and the app generates every roll number in between
- **Excluded / absent PINs** can be removed from a range
- **CSV import** by file upload, drag and drop, or paste, in two formats (PIN ranges or a full roll number list), with downloadable templates
- **Live feasibility check**: shows total, blocked and usable seats and warns when students exceed seats
- **Two result views**: colour-coded room grids and a searchable roster table filterable by room
- **Printable attendance sheet** with exam name, date, session and a signature column
- **Light / dark theme**
- **Offline fallback**: if the Prolog backend is unreachable, a built-in JavaScript backtracking solver with the same constraints is used
- Zero npm dependencies (only built-in Node.js modules)
- One-click startup on Windows with `start.bat`

---

## Seating Rules

Defined in `seat.pl`:

1. **Same class or same subject cannot sit adjacent.** Students who belong to the same class or are taking the same subject cannot occupy orthogonally adjacent seats.
2. **Adjacency** means directly left, right, in front or behind. This restriction applies only within the same examination room, and diagonal seats are allowed.
3. **No seat is shared.** Every student gets exactly one unique seat (`all_distinct`).
4. **Blocked seats are never used.**
5. **Capacity is respected.** The number of students must not exceed the usable seats (total seats minus blocked seats).

The solver uses `tuples_in/2` to restrict conflicting students to non-adjacent seat pairs, then searches with the `ffc` labeling strategy.

Seats are numbered left to right, top to bottom, starting at 1 in each room.

---

## Tech Stack

| Layer      | Technology                                  |
|------------|---------------------------------------------|
| Front end  | HTML, CSS, vanilla JavaScript               |
| Backend    | Node.js (native `http` module, ES modules)  |
| AI / Logic | SWI-Prolog with CLP(FD)                     |
| Bridge     | `prolog_bridge.pl` (JSON in, JSON out)      |

---

## Project Structure

```
AI-Based-Exam-Seating-Arrangement/
├── public/
│   ├── index.html      # User interface
│   ├── app.js          # Front-end logic, PIN engine, CSV import, fallback solver
│   └── styles.css      # Styling
├── server.js           # Node.js HTTP server and API
├── seat.pl             # CLP(FD) seating rules and solver
├── prolog_bridge.pl    # Reads JSON from Node, calls the solver, prints JSON
├── package.json        # Project metadata and start script
├── start.bat           # Windows launcher
└── LICENSE             # MIT license
```

---

## How It Works

1. The user configures rooms, blocked seats and student groups (class, subject, PIN range) in the web UI.
2. `app.js` checks capacity, builds the student list from the PIN ranges, and sends JSON to `POST /api/solve`.
3. `server.js` starts SWI-Prolog as a child process, loads `prolog_bridge.pl`, and writes the JSON to its standard input.
4. `prolog_bridge.pl` converts the JSON into Prolog terms and calls `solve/3` from `seat.pl`.
5. The solver builds the usable seat list, applies the adjacency constraints and searches for a valid arrangement.
6. The result is printed as JSON, parsed by Node.js and returned to the browser, which draws the room grids and roster table.

---

## Prerequisites

- [Node.js](https://nodejs.org/) v18 or higher
- [SWI-Prolog](https://www.swi-prolog.org/Download.html) installed on Windows
- A modern web browser

---

## Installation

```bash
git clone https://github.com/Vaishnavi022006/AI-Based-Exam-Seating-Arrangement.git
cd AI-Based-Exam-Seating-Arrangement
```

There are no npm packages to install.

### Configure the SWI-Prolog path

`server.js` uses this default path:

```js
const PROLOG_PATH = 'C:\\Program Files\\swipl\\bin\\swipl.exe';
```

If SWI-Prolog is installed somewhere else, or you are on Linux/macOS, edit this line (for example to `'swipl'` if it is on your `PATH`).

---

## Running the Project

**Option 1: Windows (easiest)**

Double-click `start.bat`. It opens `http://localhost:3000` in your browser and starts the server.

**Option 2: Manual**

```bash
npm start
```

Then open [http://localhost:3000](http://localhost:3000).

To use a different port:

```bash
set PORT=4000 && npm start        # Windows (cmd)
PORT=4000 npm start               # Linux / macOS
```

---

## Usage

1. **Exam details**: enter the institution, exam title, date and session (used on the printed sheet).
2. **Rooms**: set the name, rows and columns. Click any seat in the mini grid to toggle it as blocked `[X]`.
3. **Student groups**: enter the class name, subject, starting PIN and ending PIN. Optionally list excluded/absent PINs, separated by commas.
4. **Or import a CSV** (see below).
5. Click **Generate Seating Arrangement**.
6. Switch between the **grid view** and the **table view**, search students, then **print** the attendance sheet.

The **Load Demo** button fills in a sample (one 3x4 room with seat 5 blocked, and two groups of five students) and solves it immediately.

### CSV import formats

**Format A: PIN ranges**

```csv
Class,Subject,Start PIN,End PIN,Excluded PINs
CSE(CS),Mathematics,21B91A0501,21B91A0530,21B91A0507
ECE(EC),Physics,21B91A0401,21B91A0425,
```

**Format B: full roll number list**

```csv
Roll No,Class,Subject
21B91A0501,CSE(CS),Mathematics
21B91A0502,CSE(CS),Mathematics
21B91A0401,ECE(EC),Physics
```

Both templates can be downloaded from the import dialog. You can **replace** the current groups or **append** to them.

---

## API

### `POST /api/solve`

**Request**

```json
{
  "rooms": [
    { "room": 1, "rows": 3, "cols": 4, "blocked": [5] }
  ],
  "students": [
    { "id": "21B91A0501", "class": "CSE(CS)", "branch": "Math" },
    { "id": "21B91A0401", "class": "ECE(EC)", "branch": "Physics" }
  ],
  "groups": [
    { "class": "CSE(CS)", "branch": "Math", "count": 1 },
    { "class": "ECE(EC)", "branch": "Physics", "count": 1 }
  ]
}
```

- `rooms[].blocked` is a list of blocked seat numbers.
- `students[].branch` and `groups[].branch` represent the subject name.
- If `students` is provided, the bridge uses that list and ignores `groups`. If `students` is omitted, the bridge generates students from `groups`, with IDs like `S1_1`, `S1_2`, and so on.

**Successful response**

```json
{
  "success": true,
  "students_count": 2,
  "assignments": [
    { "id": "21B91A0501", "class": "CSE(CS)", "branch": "Math", "room": 1, "seat": 1, "row": 1, "col": 1 },
    { "id": "21B91A0401", "class": "ECE(EC)", "branch": "Physics", "room": 1, "seat": 2, "row": 1, "col": 2 }
  ]
}
```

The exact seats in your output depend on the constraints and the solver's search.

**No valid arrangement**

```json
{
  "success": false,
  "message": "No valid seating arrangement found satisfying all constraints."
}
```

**Server or Prolog error** (HTTP 500)

```json
{ "success": false, "error": "error message" }
```

### `GET /api/status`

Health check, used by the UI to show the engine badge.

```json
{ "status": "ok", "engine": "SWI-Prolog CLP(FD)" }
```

---

## Using the Prolog Solver Directly

`seat.pl` also works as a standalone command-line program:

```bash
swipl seat.pl
?- start.
```

It asks for rooms, blocked seats and student groups in the terminal and prints the seating grid.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Badge shows "Local JS/CLP Solver" | The backend is unreachable. Make sure `node server.js` is running and open the app through `http://localhost:3000`. |
| `spawn ... ENOENT` error | The path in `PROLOG_PATH` is wrong. Update it in `server.js`. |
| "No valid seating arrangement found" | Add more seats or rooms, or reduce students from the same class or subject. |
| "Not Enough Usable Seats" | Total students exceed seats minus blocked seats. |
| Port already in use | Set a different port with the `PORT` environment variable. |

---

## Screenshots

<!-- Add your screenshots to a "screenshots" folder, then uncomment the lines below.

### Home Page
![Home Page](screenshots/home.png)

### Generated Seating Arrangement
![Seating Result](screenshots/result.png)

-->

---

## Future Improvements

- Export seating plans as PDF or Excel
- Optional rule to treat diagonal seats as adjacent
- Invigilator allocation
- Save and load seating configurations

---

## Contributing

Contributions are welcome. Fork the repo, create a branch, and open a pull request.

---

## License

This project is licensed under the [MIT License](LICENSE).

---

## Author

**Vaishnavi**
GitHub: [@Vaishnavi022006](https://github.com/Vaishnavi022006)
