# Frontistirio — Scheduling & Private Lessons Brain

> **Purpose:** Documents how private lessons are tracked, how the scheduling system works with teacher/student availability, and how lesson PDFs are generated. Reference this for anything related to timetabling or lesson billing.

---

## Overview

The platform supports two lesson types:
1. **Class lessons** — a scheduled class with multiple students in a classroom (tracked via `ScheduledLesson`)
2. **Private lessons** — one-on-one (or small group) sessions tracked separately (`PrivateLesson`)

Pricing is handled via **hourly rates** at the store level, with optional **per-student overrides**.

---

## Models

### PrivateLesson (`models/private-lessons.js`)

```
PrivateLesson {
  store_id    → Store
  start_time  Date (required)
  end_time    Date (required)
  duration    Number (hours)
  course_id   → Course
  student_id  → Student
  period_id   → TeachingPeriod
  isDeleted   Boolean
  deletedAt   Date
  createdBy / updatedBy → User
}
```

### ScheduledLesson (`models/scheduledLesson.js`)

Used by the scheduler for recurring class lessons:

```
ScheduledLesson {
  store_id      → Store
  period        → TeachingPeriod
  dayOfWeek     Number (1=Mon ... 7=Sun)
  startTime     String "HH:mm"
  endTime       String "HH:mm"
  durationSlots Number
  lessonType    Enum["CLASS", "PRIVATE"]

  // For CLASS lessons:
  class_id      → ClassModel

  // For PRIVATE lessons:
  student_ids   → [Student]

  course_id     → Course (required)
  teacher_id    → Teacher (required)
  room_id       → Room (required)
  locked        Boolean
}
```

---

## API Endpoints — Private Lessons (`routes/private-lessons.js`)

| Method | Path | Description |
|---|---|---|
| GET | `/private-lessons/student/:studentId` | Get lessons for a student (date range) |
| POST | `/private-lessons/` | Create a private lesson |
| PUT | `/private-lessons/:id` | Edit a private lesson |
| DELETE | `/private-lessons/:id` | Delete a private lesson |
| GET | `/private-lessons/student/:studentId/pdf` | Generate & upload PDF of lessons |

---

## Private Lesson Creation

```
POST /private-lessons/
Body: {
  student_id, course_id, start_time, end_time, duration
}
```

1. Validate `store_id` and `period_id` (from default period)
2. Call `privateLessonHandler.createPrivateLesson(...)`
3. Lesson stored with timestamps for billing

---

## Querying Private Lessons

```
GET /private-lessons/student/:studentId?start=<ISO>&end=<ISO>
```
- Fetches lessons for a student within a date range
- Scoped to the current default period
- Used by the frontend calendar/schedule view

---

## Pricing — Hourly Rates

### Store-Level Rates (`models/hourly_rates.js`)
Default rates per course for the store:
```
HourlyRate {
  store_id    → Store
  course_id   → Course
  rate        Number (€/hour)
  period_id   → TeachingPeriod
}
```

### Student-Level Overrides (`models/student_hourly_rates.js`)
Per-student pricing override:
```
StudentHourlyRate {
  store_id    → Store
  student_id  → Student
  course_id   → Course
  rate        Number (€/hour)
  period_id   → TeachingPeriod
}
```

**Pricing resolution:** When calculating cost for a private lesson:
1. Check for `StudentHourlyRate` for this student + course → use if found
2. Fall back to `HourlyRate` for the store + course

### API Endpoints

| Route | Description |
|---|---|
| `POST /pricing-settings/` | Set store hourly rate |
| `PUT /pricing-settings/:id` | Update store rate |
| `GET /pricing-settings/` | Get all store rates |
| `POST /student-pricing-settings/` | Set student-specific rate |
| `PUT /student-pricing-settings/:id` | Update student rate |

---

## Availability Constraints

When scheduling lessons (manual or automated):

**Teacher availability** (`Teacher.period_availability`):
- Defined as workdays + time ranges per period
- e.g., Teacher available Mon–Fri 16:00–20:00

**Student unavailability** (`Student.period_unavailability`):
- Defined as blocked days + time ranges per period
- e.g., Student blocked Wed 17:00–19:00

The scheduling logic must check both before placing a lesson.

---

## PDF Generation for Student Lessons

**Endpoint:** `GET /private-lessons/student/:studentId/pdf?start=<ISO>&end=<ISO>`

Flow:
1. Fetch all private lessons for student in date range
2. Call `createStudentLessonsPdf(lessons, studentData)` (uses `pdfkit` + DejaVu fonts from `/assets/fonts/`)
3. Upload generated PDF to **AWS S3**
4. Return a **pre-signed S3 URL** (expires in 60s) for the frontend to download

Helper: `helpers/createStudentLessonsPdf.js`

---

## Teacher Availability Normalization

`normalizeWorkdays(workdays)` in `controllers/handlers/teacher.js`:
- Validates `dayOfWeek` is 1–6
- Validates `startTime`/`endTime` format (`HH:mm`)
- Deduplicates day entries
- Called before saving availability to DB
