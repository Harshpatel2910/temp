- # 🗓️ Current Approach for Handling Calendar Recurring Events

  ## 📥 Fetch Event

  - For events that we get from a particular time range:

    - We check if `recurringConfig` is present.
      - If present, we expand those recurrence events.
    - Apart from `recurringConfig` check, we also check for `parentEventId` because:
      - If `parentEventId` is present, then it is a **single recurrence event** (an exception created when the user chooses **"Update this event only"**).
      - For such single recurrence events, we **do not want to expand** the recurrence config.
      - We just can't remove `recurringConfig` from this single recurrence event because:
        - When the user chooses **"Update this and following"** or **"All"**, they might want the `recurringConfig` to be present.

  - For recurrence events, we also put `originalStartTime` in the output because
    - When the user wants to update an event with **"this and following"** or **"current only"**, we get the updated start time in the payload.
    - We want the `originalStartTime` so that:
      - We can put this start time in `excludedOccurrences` in case of **"current only"** event update.
      - When we expand this recurrence, we can exclude this updated event.
      - In case of a **"this and following"** update:
        - We break the recurrence into **2 parts**:
          - In the first part (parent part), we update the recurrence rule by adding `UNTIL = originalStartTime - 1`.
          - A new recurrence event is created from this updated event.

  ---

  ## ✏️ Update and Delete Cases

  ### ✅ Update

  - For **"All events" update**:
    - We just update the **parent event** with new properties.
    - When updating the parent event:
      - We need to remove `parentEventId` from the payload because we only store `parentEventId` for single recurrence events.
      - We remove `originalStartTime` as it is unnecessary in the event — we use it only for update purpose.
      - We also reset `excludedOccurrences` as we are updating all occurrences.
    - Apart from this:
      - We delete all the single recurrence events that have this `parentEventId` because we selected "All events", so now they are the same as the parent event.

  - For **"This and following" update**:
    - In this update, we have to do two things:
      - We break the recurrence into two parts.
      - In the first part (parent), we update the recurrence rule by adding `UNTIL = originalStartTime - 1`.
    - Two cases:
      - If `originalStartTime` and `parentStartTime` are **same**:
        - We just update the parent event.
        - We also remove `originalStartTime`, `parentEventId`, and `excludedOccurrences` as explained above.
      - If `originalStartTime` and `parentStartTime` are **not same**:
        - We update the parent recurrence with `UNTIL = originalStartTime - 1`.
        - We create a new recurrence event from this updated event.
    - Apart from this:
      - We also delete all the single recurrence events that have this `parentEventId` and whose `startTime` is greater than or equal to `originalStartTime`, as these events are now the same as the parent event.
      - As we selected **"This and following"** option, we have to remove all the events which are after this event.

  - For **"Current only" update**:
    - We create a new single recurrence event with updated properties and we also set `parentEventId` to this new event.
    - If it is already a single recurrence event (i.e., present in DB), then we just update the properties.
    - Otherwise, we create it.
    - We also add `originalStartTime` in `excludedOccurrences` of the parent event. as we don’t want to show this old event in recurrence of parent event.

  ---

  ### ❌ Delete

  - For **"All events" delete**:
    - We just delete the **parent event**.
    - We also delete all the **single recurrence events** that have this `parentEventId`, as we selected "All events" option, so now we have to remove them also.

  - For **"This and following" delete**:
    - In this case, we need two scenarios:
      - If **parent event start** and **event startTime** are **same**, then:
        - We just have to delete this parent event as there are no previous events.
      - If they are **not same**, then:
        - We have to update parent event recurrence rule by adding `UNTIL = eventStartTime - 1`.
    - Apart from this:
      - We delete all the single recurrence events that have this `parentEventId` and whose `startTime` is greater than or equal to `eventStartTime`, as we selected "This and following" option, so we have to remove all the events which are after this event.

  - For **"Current only" delete**:
    - We add `eventStartTime` in `excludedOccurrences` of parent event. as we don’t want to show this deleted event in recurrence of parent event.
    - Also, if it is a single recurrence event and is present in DB:
      - We have to delete this from DB also.
