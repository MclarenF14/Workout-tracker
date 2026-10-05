# Workout Tracker - Data Schema & Architecture

## Data Model Overview

The application uses a hierarchical data structure that flows from **Routine → Workout → Exercise → Sets**.

```
Routine (Template)
  ├─ id: string (UUID)
  ├─ name: string
  ├─ description: string
  ├─ exercises: Exercise[] (references only)
  ├─ createdAt: ISO 8601
  └─ isCustom: boolean

Workout (Active Session)
  ├─ id: string (UUID)
  ├─ routineId: string (nullable if ad-hoc)
  ├─ userId: string
  ├─ startedAt: ISO 8601
  ├─ completedAt: ISO 8601 (nullable)
  ├─ duration: number (seconds)
  ├─ notes: string
  ├─ exercises: WorkoutExercise[]
  ├─ totalVolume: number (lbs/kg)
  └─ status: 'active' | 'completed' | 'paused'

WorkoutExercise (Exercise during a workout)
  ├─ id: string (UUID)
  ├─ exerciseId: string
  ├─ name: string
  ├─ muscleGroup: MuscleGroup
  ├─ sets: Set[]
  ├─ notes: string
  ├─ order: number
  └─ lastSessionData: {
      ├─ weight: number
      ├─ reps: number
      ├─ rpe: number
    }

Set (Individual set within an exercise)
  ├─ id: string (UUID)
  ├─ reps: number
  ├─ weight: number
  ├─ rpe: number (Rate of Perceived Exertion, 1-10)
  ├─ isCompleted: boolean
  ├─ duration: number (seconds)
  ├─ notes: string
  └─ timestamp: ISO 8601

Exercise (Library Entry)
  ├─ id: string (UUID)
  ├─ name: string
  ├─ muscleGroup: MuscleGroup
  ├─ category: 'compound' | 'isolation'
  ├─ equipment: Equipment[]
  ├─ instructions: string
  ├─ alternatives: Exercise[]
  └─ imageUrl: string (optional)

MuscleGroup Type:
  'chest' | 'back' | 'shoulders' | 'biceps' | 'triceps' | 
  'forearms' | 'core' | 'glutes' | 'quadriceps' | 'hamstrings' | 'calves'

Equipment Type:
  'barbell' | 'dumbbell' | 'machine' | 'cable' | 'bodyweight' | 'kettlebell'
```

## Complete JSON Examples

### Example: Routine
```json
{
  "id": "routine-001",
  "name": "Push Day",
  "description": "Upper body pushing focus",
  "exercises": [
    {
      "id": "ex-001",
      "name": "Bench Press",
      "muscleGroup": "chest",
      "equipment": ["barbell"]
    },
    {
      "id": "ex-012",
      "name": "Incline Dumbbell Press",
      "muscleGroup": "chest",
      "equipment": ["dumbbell"]
    }
  ],
  "createdAt": "2026-10-01T10:00:00Z",
  "isCustom": true
}
```

### Example: Active Workout
```json
{
  "id": "workout-2026-10-05-001",
  "routineId": "routine-001",
  "userId": "user-123",
  "startedAt": "2026-10-05T15:30:00Z",
  "completedAt": null,
  "duration": 1825,
  "notes": "Felt strong today",
  "status": "active",
  "totalVolume": 3250,
  "exercises": [
    {
      "id": "workout-ex-001",
      "exerciseId": "ex-001",
      "name": "Bench Press",
      "muscleGroup": "chest",
      "order": 1,
      "notes": "",
      "lastSessionData": {
        "weight": 225,
        "reps": 5,
        "rpe": 8
      },
      "sets": [
        {
          "id": "set-001",
          "reps": 5,
          "weight": 225,
          "rpe": 6,
          "isCompleted": true,
          "duration": 45,
          "notes": "",
          "timestamp": "2026-10-05T15:31:00Z"
        },
        {
          "id": "set-002",
          "reps": 5,
          "weight": 225,
          "rpe": 7,
          "isCompleted": true,
          "duration": 48,
          "notes": "",
          "timestamp": "2026-10-05T15:33:30Z"
        },
        {
          "id": "set-003",
          "reps": 3,
          "weight": 225,
          "rpe": 9,
          "isCompleted": false,
          "duration": 0,
          "notes": "Feeling tired",
          "timestamp": null
        }
      ]
    }
  ]
}
```

## State Management Flow

```
┌─────────────────────────────────────────┐
│      Zustand Store (WorkoutStore)       │
├─────────────────────────────────────────┤
│  State:                                  │
│  • currentWorkout: Workout | null        │
│  • workoutHistory: Workout[]             │
│  • routines: Routine[]                   │
│  • exerciseLibrary: Exercise[]           │
│  • muscleFatigue: { [group]: number }    │
│                                          │
│  Actions:                                │
│  • startWorkout(routineId?)              │
│  • completeWorkout()                     │
│  • addExerciseToWorkout(exercise)        │
│  • addSetToExercise(exerciseId, set)     │
│  • updateSet(setId, updates)             │
│  • removeExercise(exerciseId)            │
│  • suggestAlternatives(exerciseId)       │
│  • saveToLocalStorage()                  │
│  • loadFromLocalStorage()                │
└─────────────────────────────────────────┘
         ↓
    LocalStorage (Auto-save)
         ↓
┌─────────────────────────────────────────┐
│   Backend API (Future Enhancement)      │
│   POST /workouts                        │
│   PUT /workouts/:id                     │
│   POST /routines                        │
│   GET /exercises                        │
└─────────────────────────────────────────┘
```

## Key Design Decisions

1. **Separation of Concerns:** Exercise templates are separate from workout instances, allowing history tracking without data duplication.

2. **Ghost Text Support:** `lastSessionData` is stored in `WorkoutExercise` to display previous performance without querying entire history during active workouts.

3. **Auto-save Strategy:** The Zustand store persists to LocalStorage after every meaningful action (add set, mark complete, etc.).

4. **Muscle Fatigue Calculation:** Tracked at the store level by computing recent workout volume per muscle group, enabling smart exercise substitution suggestions.

5. **RPE & Duration:** Included in Set model for detailed analytics and progression tracking.

## Next Steps

- Implement Zustand store with persistence plugins
- Build Exercise Library component with search/filter
- Create active Workout Logger with timer and real-time state updates
- Add Analytics dashboard pulling from workout history
- Implement social features (feed, routine sharing)
