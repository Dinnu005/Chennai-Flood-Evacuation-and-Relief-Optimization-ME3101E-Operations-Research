# Chennai Flood Evacuation and Relief Optimization
A Flask web app that allocates evacuation routes to shelters with capacity constraints and simulates emergency vehicle dispatch.

**Live demo:** https://or-porject-website.onrender.com/  
> Hosted on a free tier; first load can take about 30–60 seconds.

## 1) Overview
This project models evacuation support for Chennai zones during flooding. For each request, the app decides which shelter to assign and whether a shelter-based vehicle can serve the group.

Current code path:
- `app.py` loads a directed network from `data/chennai_network.json`
- `/` serves `templates/index.html`
- Frontend calls Flask APIs for allocation, metadata, and transport assignment

The implemented approach is shortest-path routing (NetworkX Dijkstra over edge weights), capacity-aware shelter assignment, and a tiered vehicle-dispatch rule engine.

## 2) Features
- Capacity-aware shelter selection (chooses the nearest *reachable* shelter with enough remaining capacity)
- Real-time shelter occupancy updates after each allocation
- 3-tier vehicle dispatch logic (available vehicle, queued wait, or no-fit alert)
- Color-coded user feedback in UI:
  - Green: dispatched / proceed with own transport
  - Yellow: all vehicles busy, wait + ETA shown
  - Red: no vehicle can fit group size
- Input validation (name, phone, location, people count)
- Unreachable-route fallback handling (`NetworkXNoPath` paths are skipped)

## 3) How it works
### Routing and shelter allocation
- The backend builds an `nx.DiGraph` from JSON nodes/arcs.
- Arc distance is converted to travel time using default speed 30 km/h.
- Missing reverse edges are added to make travel effectively bidirectional where absent in data.
- For each request group, shelters are sorted by shortest-path cost and the first shelter with enough remaining capacity is selected.

### Vehicle dispatch tiers
1. **Immediate dispatch:** smallest currently available vehicle at the assigned shelter with capacity >= group size.  
   ETA = 10-minute prep + travel time from shelter to user.
2. **All busy fallback:** if none are available, pick the engaged capable vehicle that becomes free earliest and show total ETA.
3. **No-fit fallback:** if no vehicle at that shelter can hold the group, return a “arrange own transport” message.

### UI status semantics
- **Green**: vehicle dispatched (or user chose own transport)
- **Yellow**: all capable vehicles busy; wait time + total ETA shown
- **Red**: no capable vehicle for that group size

## 4) Optimization formulation
### Transportation model (LP)
$$
\min \sum_i \sum_j c_{ij} x_{ij}
$$
subject to
$$
\sum_j x_{ij} = d_i \quad \forall i
$$
$$
\sum_i x_{ij} \le rcap_j = cap_j - occ_j \quad \forall j
$$
$$
x_{ij} \in \mathbb{Z}_{\ge 0}
$$
where:\
- $c_{ij}$ = shortest travel time (minutes) from zone $i$ to shelter $j$ from Dijkstra\
- $d_i$ = evacuees in zone $i$\
- $x_{ij}$ = evacuees assigned from zone $i$ to shelter $j$

### Assignment model (vehicle dispatch)
$$
\min \sum_{v,k} ETA_{vk} \cdot y_{vk}
$$
subject to
$$
\sum_v y_{vk} = 1 \quad \forall k
$$
$$
y_{vk} = 0 \quad \text{if } cap_v < group_k
$$
plus one-active-request-at-a-time vehicle availability constraints.

### Implementation status
| Component | Status |
|---|---|
| Dijkstra routing on directed network | **IMPLEMENTED IN THE APP** |
| Capacity-aware shelter choice | **IMPLEMENTED IN THE APP** |
| Tiered vehicle dispatch logic | **IMPLEMENTED IN THE APP** |
| LP transportation solver | **FORMULATED IN THE PROJECT REPORT ONLY** |
| Hungarian assignment solver | **FORMULATED IN THE PROJECT REPORT ONLY** |

## 5) Data / simulated network
- Data source: `data/chennai_network.json` (simulated, not live city feed)
- Nodes: 61 total = 51 evacuation zones + 10 shelters
- Arcs in JSON: 140 directed connections (backend adds missing reverse links at load time)
- Shelter capacity: 5,000 each
- Vehicles: 50 total (5 per shelter), capacities sampled from 10 / 20 / 50 seats
- Initial occupancy: random 1,000–4,500 for each shelter, with `s1` (Guindy) forced to 4,980 for stress testing

## 6) Example walkthrough (Mylapore, group size 25)
Verified shortest path from `n5` (Mylapore) to `s10` (Nungambakkam) in current code/data:
- Path: **Mylapore → Alwarpet → Teynampet → Shelter 10 (Nungambakkam)**
- Route length: **4.1 km**
- Estimated travel time (routing cost): **8.2 minutes**

Capacity behavior:
- The allocator checks shelters in increasing shortest-path order and assigns the first with enough room.
- Occupancy values are randomized at startup, so exact “before/after” counts vary run to run.
- If S10 had occupancy 2,340 at that moment, assigning 25 would reduce remaining capacity to 2,315 occupied-equivalent accounting as expected.

<img src="docs/images/placeholder-route-1.png" alt="Route screenshot placeholder" />
<img src="docs/images/placeholder-status-1.png" alt="Status screenshot placeholder" />

## 7) Test scenarios covered
- Routine evacuation request (e.g., 10 people from Mylapore)
- Near-capacity shelter behavior (redirection when a candidate shelter cannot fit a group)
- Burst traffic behavior (many sequential requests consume capacity and vehicles)
- Self-transport flow (`Need transport? = No`)
- Input validation failures (invalid phone, missing inputs)
- Unreachable-route handling (`NetworkXNoPath` candidates skipped)

## 8) Limitations and future work
- Static network and demand (no time-dependent congestion)
- Reduced-scale network vs full city graph
- Single-trip vehicle assignment (no multi-stop VRP)
- No uncertainty modeling (closures/demand surges)
- No shelter queueing dynamics (queueing/discrete-event simulation absent)
- Full-scale city optimization with dedicated solvers (e.g., Gurobi/CP-SAT) not implemented
- No live external data integration yet (traffic/rainfall/real occupancy)

## 9) Run locally
Verified commands:
```bash
pip install -r requirements.txt
python app.py
```
App URL: `http://127.0.0.1:5000/`

Gunicorn deployment command (also verified to boot):
```bash
gunicorn app:app
```
Default bind is `http://127.0.0.1:8000/` unless overridden.

## 10) Project structure
```text
.
├── app.py                     # Flask backend APIs and routing/dispatch logic
├── README.md                  # Project documentation
├── requirements.txt           # Python dependencies
├── data/
│   └── chennai_network.json   # Simulated Chennai network (zones, shelters, arcs)
├── templates/
│   └── index.html             # Frontend UI (Tailwind + vanilla JS)
└── docs/
    └── images/                # Screenshot placeholders/assets
```

## 11) Tech stack
- Python
- Flask
- Flask-CORS
- NetworkX
- Gunicorn
- HTML + Tailwind CSS (CDN) + vanilla JavaScript

## 12) Team and context
Course project for **ME3101E Operations Research**, NIT Calicut (Instructor: **Dr. Anoop K P**).

**Group 11:**
- Ganesh S
- Palli Dinesh Sai
- Tapa Ankitha
- Tangallapalli Akshay Kumar
- Thanvi Badavath

---
Fork note: this repository was forked from **ganesh23505/OR_PORJECT**.
