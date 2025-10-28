# Recap so far

- Phases: **Design → Train → Operate**.
- In **Train**, we covered **experiment tracking** and **turning notebooks into ML pipelines**.
- Output of training: a **model** to deploy.

# Operate phase: model deployment options

First question: **Do we need predictions immediately?**

- **If we can wait** (minutes/hours/days/weeks) → **Batch mode** (also called batch/“offline” deployment).
- **If we need them now** → **Online deployment**, with two variants:
    - **Web service**
    - **Streaming**

# Batch mode (apply model on a schedule)

- Run at **regular intervals** (e.g., every 10 min / 30 min / hour / 6 hours / day / week / month).
- Typical flow:
    1. **Database** holds the data.
    2. A **scoring/prediction job** (contains the model) **pulls data** from the last interval (e.g., “yesterday” if daily; “previous hour” if hourly).
    3. Job **writes predictions** to another place (e.g., a predictions DB).
    4. **Something else** reads predictions and reacts (e.g., a **report**).
- **Example (marketing / churn):** Taxi app users might switch (e.g., to Uber).
    - Daily job scores **who is about to churn**.
    - A **marketing/reward job** reads these predictions and **sends pushes**.
    - Doesn’t need to run all the time—**daily is fine**.

# Online deployment — Web service

- Model is **up and running all the time**; backend **calls it over HTTP** and gets a prediction.
- **1-to-1, synchronous**: client (backend) ↔ model service; **connection stays alive** until response.
- **Example (ride duration):** App/backend sends features (user id, pickup/dropoff, time, etc.) to a **ride-duration service**; it returns **“30 minutes”** so the user can decide **now**.
    - Needs **immediate** response on the booking screen.

# Online deployment — Streaming

- **Producers** publish **events** to a **stream**; **consumers** read and react **independently**.
- **1-to-many / many-to-many**, **no direct request/response**; the producer **doesn’t care who** consumes.
- Taxi flow: backend becomes a **producer** on “**ride_started**” (event includes user id, pickup/dropoff, etc.).
    - **Consumers** might include:
        - **Tip prediction** service (maybe triggers a push).
        - **More accurate duration** model after the ride starts (can **update** ETA: e.g., from 30 → 20 minutes).
- **Content moderation example (YouTube-style):**
    - On “**video_uploaded**” event, multiple consumers check **copyright**, **NSFW**, **violence/hate**, etc.
    - Each pushes its **predictions** to a **predictions stream**.
    - A **decision/moderation service** aggregates: if a consumer flags removal (with reason), it **removes the video**.
- **Recommendations example:** On “**video_published**”, a component decides **who to notify / which feeds to update**.
