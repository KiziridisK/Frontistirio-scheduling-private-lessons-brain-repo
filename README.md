# Frontistirio — Scheduling & Private Lessons Brain

> **Full-stack reference** for private lesson tracking, hourly rate pricing (store-level + per-student overrides), PDF billing export, and the availability constraint system. Covers backend models, routes, controller logic, and the Angular frontend.

---

## Backend: Models

### `models/private-lessons.js`
```javascript
{
  store_id: ObjectId → Store (required),
  start_time: Date (required),
  end_time: Date (required),
  duration: Number (hours, required, min: 0),
  course_id: ObjectId → Course (required),
  student_id: ObjectId → Student (required),
  period_id: ObjectId → TeachingPeriod (required),
  isDeleted: Boolean, deletedAt: Date,
  createdBy/updatedBy: ObjectId → User,
  timestamps: true
}
```

### `models/scheduledLesson.js` (for future scheduler)
```javascript
{
  store_id: ObjectId → Store,
  period: ObjectId → TeachingPeriod,
  dayOfWeek: Number (1–7),
  startTime: String 'HH:mm', endTime: String 'HH:mm',
  durationSlots: Number,
  lessonType: Enum['CLASS', 'PRIVATE'],
  class_id: ObjectId → ClassModel (for CLASS type),
  student_ids: [ObjectId → Student] (for PRIVATE type),
  course_id: ObjectId → Course (required),
  teacher_id: ObjectId → Teacher (required),
  room_id: ObjectId → Room (required),
  locked: Boolean  // prevents auto-scheduler from moving this slot
}
```

### `models/hourly_rates.js` (store-level rates)
```javascript
{
  store_id: ObjectId → Store,
  course_id: ObjectId → Course,
  period_rates: [{
    period: ObjectId → TeachingPeriod,
    rate: Number (€/hour)
  }],
  isDeleted: Boolean
}
```

### `models/student_hourly_rates.js` (per-student overrides)
```javascript
{
  store_id: ObjectId → Store,
  student_id: ObjectId → Student,
  course_id: ObjectId → Course,
  period_rates: [{
    period: ObjectId → TeachingPeriod,
    rate: Number (€/hour)
  }],
  isDeleted: Boolean
}
```

---

## Backend: Routes

### `/private-lessons` (`routes/private-lessons.js`)
| Method | Path | Role | Controller |
|---|---|---|---|
| GET | `/private-lessons/get-student-private-lessons/:studentId` | isStoreUser | `getStudentPrivateLessons` |
| POST | `/private-lessons/create-student-private-lesson` | isStoreUser | `createStudentPrivateLesson` |
| POST | `/private-lessons/export-student-private-lessons` | isStoreUser | `exportStudentPrivateLessons` |
| DELETE | `/private-lessons/delete-student-private-lesson` | isStoreUser | `deleteStudentPrivateLessons` |

### `/pricing-settings` (hourly_rates routes)
| Method | Path | Role |
|---|---|---|
| GET | `/pricing-settings/get-store-pricing-settings` | isStoreUser |
| POST | `/pricing-settings/upsert-store-pricing-settings` | isStoreUser |
| DELETE | `/pricing-settings/delete-store-pricing-setting` | isStoreUser |

### `/student-pricing-settings` (student_hourly_rates routes)
| Method | Path | Role |
|---|---|---|
| GET | `/student-pricing-settings/get-student-pricing-settings` | isStoreUser |
| POST | `/student-pricing-settings/upsert-student-pricing-settings` | isStoreUser |

---

## Backend: Private Lesson Query Logic

### `getStudentPrivateLessons(student_id, start, end, periodId)`
```javascript
PrivateLesson.find({
  student_id: student_id,
  period_id: periodId,
  isDeleted: false,
  start_time: { $gte: new Date(start), $lte: new Date(end) }
})
```
Filtered by: student, period, date range. Returns all lessons (no populate by default).

### `createStudentPrivateLesson` (with MongoDB Transaction)
Validates: `store_id`, `periodId`, `course_id`, `student_id`, `start_time`, `end_time`, `duration`. Creates `PrivateLesson` document inside a session.

---

## Backend: Export PDF — Full Logic (`exportStudentPrivateLessons`)

This is the most complex controller in the private lessons module:

**Step 1 — Fetch lessons:**
```javascript
// body: { student_id, date_from, date_to }
const student = await studentHandler.getStudentById(student_id);
const lessons = await privateLessonHandler.getStudentPrivateLessons(
  student_id, date_from, date_to, periodId
);
```

**Step 2 — Fetch rates in parallel:**
```javascript
const rate_promises = [];
rate_promises.push(studentHourlyRatesHandler.getStoreStudentHourlyRates(store_id, student_id, periodId));
rate_promises.push(hourlyRatesHandler.getStoreHourlyRates(store_id, periodId));
const [student_rates, store_rates] = await Promise.all(rate_promises);
```

**Step 3 — Rate resolution per course (priority logic):**
```javascript
// For each unique course_id in the lessons:
// 1. Look for student-specific rate for this course + period
// 2. If not found → fall back to store-level rate for this course + period
// 3. Result: rates_to_use = { [course_id]: rate_per_hour }
const rates_to_use = {};
lesson_course_ids.forEach(id => {
  // find in student_rates first, then store_rates
  // only assigns if period_rate.rate > 0
});
```

**Step 4 — Generate PDF:**
```javascript
const fileContent = await createStudentLessonsPdf(
  student, lessons, rates_to_use, date_from, date_to, courses
);
```

**Step 5 — Upload to S3:**
```javascript
// Filename: "FirstName_LastName_DD-MM-YYYY_HH:mm_DD-MM-YYYY_HH:mm.pdf"
// S3 path: "private-lesson-costs/{group_id}/{store_id}/{timestamp}_{filename}"
// Bucket: "logeion-private-lesson-costs" (hardcoded, different from educational material bucket)
const command = new PutObjectCommand({ Bucket, Key: s3Key, Body: fileContent, ContentType: 'application/pdf' });
await s3Client.send(command);
```

**Step 6 — Return pre-signed URL (60s):**
```javascript
const signedUrl = await getSignedUrl(s3Client, new GetObjectCommand({
  Bucket, Key: s3Key,
  ResponseContentDisposition: `attachment; filename="${encodedFilename}"`,
  ResponseContentType: 'application/pdf'
}), { expiresIn: 60 });
res.json({ success: true, downloadUrl: signedUrl, filename });
```

---

## Backend: Hourly Rates — Aggregation Query

`getStoreHourlyRates(storeId, periodId)` uses aggregation to filter `period_rates` to the current period:
```javascript
HourlyRates.aggregate([
  { $match: { store_id: storeOid, isDeleted: false } },
  { $addFields: {
    period_rates_filtered: {
      $filter: { input: '$period_rates', as: 'pr',
        cond: { $eq: ['$$pr.period', periodOid] } }
    }
  }},
  { $match: { 'period_rates_filtered.0': { $exists: true } } }
  // Only returns rates that have an entry for this period
])
```

---

## Frontend: TypeScript Model

```typescript
interface PrivateLesson {
  _id?: string; store_id?: string;
  start_time: Date | string; end_time: Date | string;
  duration: number;       // hours
  course_id: string; student_id: string; period_id?: string;
  isDeleted?: boolean;
}
```

---

## Frontend: Service (`services/private-lessons.service.ts`)

```typescript
getStudentPrivateLessons(studentId, start, end)
  → GET /private-lessons/get-student-private-lessons/:studentId
    ?start=<ISO>&end=<ISO>
  → returns { success, message, lessons: PrivateLesson[] }

addStudentPrivateLesson(lesson, student_id)
  → POST /private-lessons/create-student-private-lesson
  body: { lesson: { course_id, start_time, end_time, duration }, student_id }

editStudentPrivateLesson(lesson)
  → POST /private-lessons/edit-store-class   ⚠️ misnamed endpoint

deleteStudentPrivateLesson(lessonId, permanently)
  → DELETE /private-lessons/delete-student-private-lesson
  body: { id: lessonId, permanently }

exportStudentLessonPrices(payload)
  → POST /private-lessons/export-student-private-lessons
  body: { student_id, date_from, date_to }
  → returns { success, downloadUrl, filename, message }
```

### `PriceSettingsService` (`services/price-settings.service.ts`)
```typescript
getStoreHourlyRates()
  → GET /pricing-settings/get-store-pricing-settings

upsertStoreHourlyRate(rate)
  → POST /pricing-settings/upsert-store-pricing-settings

getStudentHourlyRates(studentId)
  → GET /student-pricing-settings/get-student-pricing-settings?studentId=xxx

upsertStudentHourlyRate(rate)
  → POST /student-pricing-settings/upsert-student-pricing-settings
```

---

## Frontend: Components

### `StudentDetailsComponent` (manages lessons)
Embedded within the student detail view:
- Date range picker for lesson calendar
- List of private lessons with edit/delete per lesson
- "Add Lesson" form: course selector, date/time, duration
- "Export Billing" button → opens `ExportStudentLessonPricesComponent`

### `ExportStudentLessonPricesComponent` (`students/export-student-lesson-prices/`)
```typescript
// Date range picker → calls:
PrivateLessonsService.exportStudentLessonPrices({
  student_id: student._id,
  date_from: selectedStart,
  date_to: selectedEnd
}).subscribe(res => {
  if (res.success) window.open(res.downloadUrl, '_blank');
});
```

### `SetRatesComponent` (`prices/set-rates/`)
UI for managing store-level hourly rates per course. Part of the settings/pricing section.

---

## Availability System

### Teacher Availability (stored on Teacher document)
```javascript
period_availability: [{
  period: ObjectId,
  workdays: [{ dayOfWeek: 1–6, timeRanges: [{ startTime, endTime }] }]
}]
```
Set via `POST /teachers/upsert-teacher-details` with `changes.period_availability`.
Backend validates with `normalizeWorkdays()`.

### Student Unavailability (stored on Student document)
```javascript
period_unavailability: [{
  period: ObjectId,
  blockedDays: [{ dayOfWeek: 1–7, timeRanges: [{ startTime, endTime }] }]
}]
```
Set via `POST /students/upsert-student-unavailability`.

Both are **stored but not yet enforced automatically** — the platform currently uses manual booking. The `ScheduledLesson` model exists for a future automated timetabling feature.

---

## S3 Bucket Notes

Private lesson PDFs use a **different bucket** (`logeion-private-lesson-costs`) from educational materials (`logeion-educational-material`). The path is:
```
private-lesson-costs/{group_id}/{store_id}/{timestamp}_{StudentName}_{from}_{to}.pdf
```

Files are not automatically deleted — they accumulate in S3 unless cleaned up manually.
