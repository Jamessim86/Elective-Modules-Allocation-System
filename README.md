# Elective Modules Allocation System

A single-page web app for the School of Business & Accountancy's PET elective module allocation exercise. Everything runs in the browser, with no server, no external AI service and no data upload. The same inputs always give the same allocation.

## Files

| File | Purpose |
|---|---|
| `index.html` | The complete app. The Excel reader/writer (SheetJS, Apache-2.0) and the allocation engine are bundled inside, so it also works offline. |
| `sample_students.xlsx` | 800 demo students in the upload format. Use it with **Load demo settings** to try the app. |

## Deploy on GitHub Pages

1. Create a new repository on GitHub, e.g. `elective-allocation`. A private repo works if your plan supports private Pages.
2. Upload `index.html` (and this README if you want it) to the root of the default branch.
3. Go to **Settings → Pages**. Under *Build and deployment*, choose **Deploy from a branch**, select `main` and `/ (root)`, then **Save**.
4. After about a minute the app is live at `https://<your-account>.github.io/elective-allocation/`.

You can also just double-click `index.html` and it opens and runs locally in Chrome or Edge.

> Student data and settings are saved only in the browser's local storage on the computer you use. Use **Export settings** to back up or share a configuration.

## Workflow

1. **Diplomas & Modules**: choose the participating diplomas, each diploma's predetermined semester (April/September), how many electives each diploma needs, and the module-by-diploma availability matrix. You can add or remove diplomas and modules.
2. **Class Sizes & Rules**:
   - Master min/ideal/max class size (default 20/25/27), with optional overrides per module.
   - Optional number of classes per module and semester. Blank means the AI decides. `0` means the module isn't offered that semester. Any other number is a hard constraint.
   - A top-N preference slider.
   - Whether to include students flagged as potential disruption.
   - The tie-break order.
3. **Student Data**: upload the Excel file. Expected columns:
   - `Email`, `Name`, `Diploma` (code or full name), `Current Class`
   - `Ranking`: modules separated by `;`, most preferred first. Leave it blank if the student didn't submit.
   - `Potential Disruption`: `Yes` or blank.
   - An optional `Semester` column overrides the diploma's semester for that student.

   The app flags these errors before allocation: duplicate identifiers, a module ranked twice, unrecognised diplomas or modules, and missing columns.
4. **Allocate & Results**:
   - **Overview**: key figures, the preference distribution chart (4.1), recommendations and the AI's decisions.
   - **Student Allocation**: report 4.2, with filters, CSV export and a manual **Edit** for each student.
   - **Class Utilisation**: report 4.3, with ± buttons to change the number of classes, a CSV export and a class lists export.
   - **Unmet Needs**: report 4.4, with a reason and recommended parameter changes for each case.
   - **Disruption & Non-submitters**: report 4.5, including each class's size if the flagged students withdraw.
   - **Anomalies**: report 4.6.
   - **Export all (Excel)**: puts every report into one workbook.

## How the allocation engine works

The engine solves a **minimum-cost network flow**. Students who are identical in every respect are grouped together. Each group connects to every module it can take in its semester, and every module connects to the sink through class-size capacities. The requirement priorities in section 3.9 are encoded as cost tiers that differ by orders of magnitude, so each tier strictly dominates the ones below it:

| Priority | How it is enforced |
|---|---|
| (a) Hard constraints | Eligibility comes from the graph structure: there is simply no edge to an unavailable module. The required number of modules is the supply per student. Each class's minimum gets a very large reward for being filled and its maximum is an edge capacity. User-fixed class counts are kept fixed. |
| (b) Ranked before non-submitters | Non-submitters cost the same in every module, so they never take a preferred place from a ranked student. If seats run out, they are the first to go unallocated. |
| (c) Top-N preferences | Cost rises with each rank step. A large jump applies outside the top N, and a larger one for modules the student did not rank. Rankings of modules that are unavailable or not opened are skipped. The next available choice takes their place, but reports show the original rank. |
| (d) Potential-disruption students | They count towards a class's minimum only at a penalty, so classes are filled to minimum by students who will stay. |
| (e) Ideal class size | A small penalty applies for each student above the ideal size. |

A deterministic search around the flow decides how many classes each module opens:

1. Measure the natural demand for each module.
2. Repeatedly take the least-subscribed module that is below minimum. Demand counts only students who ranked the module; non-submitters never justify opening a class on their own. Either close it or keep it with one class topped up to the minimum, whichever gives the better overall outcome.
3. Size each open module to the class count closest to the ideal size.
4. Add a class wherever blocked demand justifies one.

Every decision is logged on the Overview tab. Modules that don't open show the reason on the Class Utilisation tab.

**Fairness.** Students with identical rankings, diploma, semester and flags are treated as one group. If there are not enough places for all of them, the places go by the chosen tie-break order: file row order, or a reproducible random order seeded by email. Every such case is flagged under **Anomalies** for manual review.

**Single-course classes (optional).** On the Class Sizes & Rules tab you can tick modules whose classes must contain one course only, and courses whose students must only be taught with their own course. A class must be single-course if either rule applies. With nothing ticked, the default, courses may share classes and results are unchanged. When the rule applies it is strict: each course is planned as its own group, with the same minimum and maximum class sizes, and students who can't form a class move to another choice. A fixed number of classes stays a total for the module. The AI splits it across courses and, if the total can't work, flags it and recommends a class count that does.

**Number of classes per module.** On the Class Sizes & Rules tab, each module has two optional fields per semester. **At least N** means the AI must open N or more classes, but it won't open a class it can't fill to the minimum size; if N can't be reached, it says so and suggests the highest number that works. **Exactly N** is a hard rule, and 0 means the module isn't offered. Fill in only one of the two; the app asks you to correct it if both are filled.

**Reserved classes (optional).** These rules are for classes that must be held regardless of demand. Each rule names a module, the courses it applies to (or All courses), a type, whether other courses may join, and optionally a number of classes (at least or exactly).
- **Hold class(es):** the AI must open the class and fill it with students from the selected courses, starting with those who ranked it highest. If other courses may join, they only fill spare places.
- **Compulsory:** every student of the selected courses takes the module, and it counts as one of their required electives. Compulsory allocations are left out of the preference statistics.

A rule's classes count inside the module's total. Each rule shows on the Class Utilisation tab as met or not met, with the reason and a suggested fix. Rules that aren't met also appear under Anomalies and Recommendations, and on the Overview. Each run covers one semester's student file, so rules apply to that semester.

**Backup candidates (Disruption & Non-submitters tab).** For every place held by a potential-disruption student, the app lists up to 3 backups. These are students who ranked that module higher than a module they currently have, and who pass the hard rules: course availability, semester, and single-course or reserved-class rules. Backups are listed in this order:
1. students on the Unmet Needs list, labelled "Unmet needs",
2. other students who would improve, labelled "Would improve",
3. potential-disruption students, marked "may also withdraw".

A warning shows if the class a backup would leave would drop below its minimum. Use **Withdraw** to mark a student as withdrawn and free their place(s), then **Promote** to move a backup in. The backup gives up their lowest-ranked module, never a compulsory one. Both steps can be undone, every report updates, and the moves are recorded as manual adjustments. Backups are worked out after the allocation and never change it.

**Class groups (optional).** These keep a group of students from one course together as one class, in classes of their own. The group's size stays within the class-size settings of the module(s), between the minimum and maximum. When you set a number of groups, the AI tries each size in that range and keeps the one with the best overall result. A size that leaves the module's other students a whole number of classes avoids rebuilding the rest of the allocation. With "all students of the course", the course is split evenly into groups within the range. Each rule names a course, either a number of groups or "all students of the course", and what the group shares:
- **Both modules, one after another.** For courses whose students take exactly 2 electives. The first module runs in **Block A** (about weeks 1–7) and the second in **Block B** (about weeks 8–15). The module pair and the block order can be left to the AI or chosen.
- **One module, together over the full semester.** For any course. Each member's other elective is allocated as usual from their own rankings. Other students, including others from the same course, can still take that module in normal classes. The module can be left to the AI or chosen.

The AI tests the most promising module(s) through the full allocation and keeps the option with the best overall result. It fills each group first with students who would have had the module(s) anyway, then with those who ranked them highest, so a group costs as little as possible. Everyone else is allocated as usual. The results page shows what was chosen and why, how members had ranked it, and how many members would have had higher-ranked modules without the group. An **Overall effect** line compares the whole cohort with and without the groups: 1st choices, top-N as ranked, classes and, where relevant, unmet needs. Students outside a group may get different modules, because the rest of the allocation is planned around the groups. Each member's Remarks show what they would otherwise have had. Sharing one module usually costs far less than sharing both.

**Class assignment.** Within a module, students are split into near-equal classes grouped by diploma and current class. Potential-disruption students are spread out so that every class stays at or above its minimum if they withdraw.

## Notes and assumptions

- A student's electives all run in one semester: their diploma's semester, unless the file's Semester column overrides it. Classes are planned separately for April and September.
- Re-running the allocation discards manual edits. The app asks you to confirm first.
- The bundled SheetJS build is 0.18.5, the version available on npm. It's fine for trusted internal files. Avoid opening Excel files from unknown sources.
