## Scenario

I missed the bus because it rained, so I was late to class.

### Step 0) turning the sentence into “events + variables”

- Event A: Rain happened (during some time window)
- Event B: You attempted to catch a specific bus trip
- Outcome 1: You missed that bus
- Outcome 2: You arrived late to a class session
- Causal claim: Rain -> MissedBus -> LateToClass

The key modeling move: 
- reify verbs into events when you need time, context, or evidence

### Step 1) Competency questions

1. Which class was I late to, and by how many minutes?
2. Which bus trip was missed? (route, scheduled departure, actual departure)
3. When/where did it rain (or what weather observation do we have)?
4. Do we believe rain caused the miss, or is it just correlated?

### Operation level relational model

Entities:
- Person
- BusRoute
- ClassSession

Events: 
- WeatherObservation (or just a boolean “rainy”)
- CommuteAttempt
- MissedConnection (missing the bus is an event)
- Arrival (arrived to campus/class)

For example:
- Person(person_id, name)
- BusRoute(route_id, name)
- ClassSession(session_id, course_code, start_time, location)
- WeatherObservation(obs_id, ts_start, ts_end, location, - precipitation_mm, is_raining)
- CommuteAttempt(attempt_id, person_id, ts_depart_home, origin, - destination)
- BusTrip(trip_id, route_id, scheduled_departure, actual_departure, - stop_location)
- MissedBus(miss_id, attempt_id, trip_id, miss_reason, ts_recorded)
- Arrival(arrival_id, attempt_id, location, arrived_at)
- LateToClass(late_id, person_id, session_id, arrival_id, minutes_late)

Thought process behind these choices:
- “Missed the bus” is not a static attribute, it’s an event tied to a particular BusTrip.
- “Late to class” should reference a specific ClassSession (otherwise we cannot compute lateness).
- WeatherObservation is separated because it can be shared across many people/attempts and can come from different sources.

### Knowledge graph model (knowledge level)

Example triples in RDF style:
- :Attempt_2025_12_25_0805  :agent        :Person01 .
- :Attempt_2025_12_25_0805  :target       :CS101_Lecture_0900 .
- :Attempt_2025_12_25_0805  :missedTrip   :BusTrip_34A_0815 .

- :RainObs_0800_0830        :isRaining    true .
- :RainObs_0800_0830        :observedAt   :HomeArea .
- :Attempt_2025_12_25_0805  :hasContext   :RainObs_0800_0830 .

- :LateEvent_1              :agent        :Omer .
- :LateEvent_1              :forSession   :CS101_Lecture_0900 .
- :LateEvent_1              :minutesLate  12 .
- :LateEvent_1              :causedBy     :MissedBusEvent_1 .