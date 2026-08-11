# Feature Specification: Add Support for Body Temperature Metric Ingestion, Storage, and Reporting

**Feature Branch**: `TEST123PUB-127`
**Created**: 2025-08-13
**Status**: Draft
**Story**: [TEST123PUB-127](https://stg.jsw.ibm.com/browse/TEST123PUB-127)

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ingest Body Temperature from Devices and APIs (Priority: P1)

A user who owns a compatible smart thermometer or wearable device wants the Sapphire FitConnect platform to accept their body temperature readings automatically. When the device or an integrated API sends a temperature measurement, the platform validates it, associates it with the user's profile and the source device, and stores it for future retrieval — whether the data arrives one record at a time or in a batch upload.

**Why this priority**: Without the ability to accept temperature data, no downstream feature (storage, charting, or UI display) can function. Ingestion is the foundational capability that unlocks all other stories. Delivering this first provides a working data pipeline that the rest of the feature can build on.

**Independent Test**: Send a valid temperature reading via the data submission endpoint (both single-record and batch payloads). Verify the system returns a success acknowledgement, the record is retrievable via the data access layer, and the record contains the correct user, timestamp, unit, device source, and value. No UI or chart component is required to validate this story.

**Acceptance Scenarios**:

1. **Given** a registered user with a linked smart device, **When** the device sends a single temperature reading with a valid value, unit (Celsius or Fahrenheit), timestamp, and device identifier, **Then** the platform accepts the record, stores it associated with the user, and returns a successful acknowledgement.
2. **Given** a registered user, **When** an integrated API submits a batch of up to 100 temperature readings in a single request, **Then** all records are accepted, stored, and acknowledged; no partial batch is silently discarded.
3. **Given** a temperature reading is submitted, **When** the value falls outside the configurable physiological range (e.g., below minimum or above maximum allowed threshold), **Then** the platform rejects the record with a descriptive error message stating the valid range and the submitted value.
4. **Given** a temperature reading is submitted, **When** the unit is unrecognised or absent, **Then** the platform rejects the record with a clear error indicating the accepted units.
5. **Given** a temperature reading is submitted, **When** the timestamp is missing or in an invalid format, **Then** the platform rejects the record with a clear error indicating the required format.

---

### User Story 2 - View Body Temperature Trend Charts (Priority: P2)

A user navigating the health dashboard wants to see a visual chart of their body temperature history so they can understand trends over time — daily fluctuations, weekly patterns, or monthly averages. The chart displays minimum, maximum, and average values for the selected time range and clearly labels the unit in use.

**Why this priority**: Visualisation is the primary user-facing output of this feature. Once ingestion and storage are in place (P1), delivering the chart gives users immediate, tangible value and satisfies the core product requirement of making temperature data visible in the portal. This is prioritised above unit switching and export because it covers the widest user need.

**Independent Test**: With pre-loaded temperature records for a test user, open the health dashboard and navigate to the temperature chart. Select each available time range (day, week, month). Verify that: (a) the chart renders without error, (b) min/max/average values match the stored records, (c) the selected unit is clearly displayed, and (d) loading and empty-state conditions are handled gracefully.

**Acceptance Scenarios**:

1. **Given** a user has at least one stored temperature record, **When** they open the temperature section of the health dashboard and select the "Day" range, **Then** a chart displays their temperature readings for the last 24 hours with min, max, and average values labelled.
2. **Given** a user selects the "Week" or "Month" range, **Then** the chart updates to display aggregated min, max, and average temperature values for the selected period.
3. **Given** a user filters by a specific device source, **Then** the chart updates to show only readings from that device.
4. **Given** a user has no temperature records, **When** they open the temperature chart, **Then** an informative empty-state message is shown (e.g., "No temperature data recorded yet"), not an error or blank space.
5. **Given** the chart data is loading, **When** the request is in flight, **Then** a loading skeleton or indicator is displayed instead of blank space.

---

### User Story 3 - Include Temperature in Health Analytics Export (Priority: P3)

A user who exports their health data (for sharing with a healthcare provider or personal records) wants body temperature to be included in the export alongside other metrics such as blood pressure and SpO2. The export reflects the same filtered date range and device source that the user has selected in the dashboard.

**Why this priority**: Export is an important but secondary capability — it is useful only once ingestion and visualisation are working. It does not block the core health-tracking workflow, but adds significant value for users who share data with healthcare providers.

**Independent Test**: Trigger a health analytics export for a date range that includes temperature records. Open the resulting export and verify that temperature data (value, unit, timestamp, device source) appears in the output alongside other metric types. Verify that records outside the selected date range are excluded.

**Acceptance Scenarios**:

1. **Given** a user triggers a health data export for a date range, **When** temperature records exist for that range, **Then** the export includes a temperature section with value, unit, timestamp, and device source for each record.
2. **Given** a user applies a device-source filter before exporting, **When** the export is generated, **Then** only temperature records matching the selected device source are included.
3. **Given** no temperature records exist in the selected date range, **When** the export is generated, **Then** the temperature section is omitted or noted as empty — no error is produced.

---

### Edge Cases

- What happens when a batch submission contains a mix of valid and invalid temperature records? The system must reject only the invalid records and accept the valid ones, or reject the entire batch atomically — the behaviour must be explicitly defined and consistent with existing batch ingestion patterns.
- How does the system handle duplicate temperature readings (same user, same timestamp, same device, same value) submitted more than once? The system must be idempotent — duplicate submissions must not result in duplicate stored records.
- What happens if a Celsius and a Fahrenheit value for the same physiological temperature are submitted from different devices simultaneously? Both must be stored as submitted; the display layer converts to the user's preferred unit.
- What happens if the device source identifier is present but references an unknown or unregistered device? The record should be accepted with the source identifier preserved, but flagged as unverified device.
- What happens if the configurable physiological range is updated after records have been stored? Existing records outside the new range must not be retroactively deleted or hidden.
- What happens when a chart request covers a period with no data for some time sub-ranges (gaps)? The chart must render correctly with gaps, not substitute zeroes or crash.
- How does the system handle extremely high-frequency submissions (e.g., continuous monitoring devices sending readings every 30 seconds)? Rate limiting and storage capacity must be considered.

---

## Requirements *(mandatory)*

### Functional Requirements

**Data Ingestion**

- **FR-001**: The system MUST accept body temperature measurements submitted by integrated smart devices and third-party APIs via the health data submission endpoint.
- **FR-002**: The system MUST support temperature values expressed in both Celsius and Fahrenheit; the unit MUST be stored alongside the value as submitted.
- **FR-003**: Each accepted temperature record MUST be associated with: the user identifier, a timestamp (ISO 8601 with timezone), the source device identifier, the ingestion source (device push or API), and optionally the measurement method (e.g., oral, axillary, tympanic).
- **FR-004**: The system MUST validate submitted temperature values against a configurable physiological range; values outside the range MUST be rejected with an error response that specifies the valid range and the submitted value.
- **FR-005**: The system MUST accept both single-record and batch submissions; batch submissions MUST support at least 100 records per request.
- **FR-006**: The system MUST be idempotent for temperature record submissions — a duplicate submission (same user, timestamp, device, and value) MUST NOT create a duplicate stored record.

**Data Model / Schema**

- **FR-007**: The temperature metric type MUST be added to the health metrics catalogue alongside existing types (blood pressure, SpO2, activity).
- **FR-008**: The temperature schema MUST define fields for: numeric value, unit (Celsius/Fahrenheit), ISO 8601 timestamp, device source identifier, ingestion source, and optional measurement method.
- **FR-009**: The temperature schema MUST be compatible with the existing downstream analytics pipeline and timeseries storage layer.
- **FR-010**: Schema documentation MUST be updated to include the temperature metric type and field definitions.

**Storage and Processing**

- **FR-011**: Temperature records MUST be stored in the existing timeseries metrics datastore with time-based indexing that supports efficient range queries.
- **FR-012**: The storage layer MUST apply the existing retention and aggregation rules to temperature records; daily, weekly, and monthly rollups MUST be computed for temperature.

**Reporting and Charting**

- **FR-013**: The charting service MUST provide temperature trend data (minimum, maximum, and average values) for user-specified time ranges: day, week, and month.
- **FR-014**: The charting service MUST support filtering temperature trend data by date range and by device source.
- **FR-015**: Temperature metric data MUST be included in the health analytics export endpoint, honouring the same date range and device source filters.
- **FR-016**: Temperature data MUST appear in the user's overall health analytics dashboard alongside other metrics.

**User Interface**

- **FR-017**: The temperature metric MUST appear in the user's metrics list in the health dashboard.
- **FR-018**: The UI MUST provide a temperature chart component displaying trend data (min, max, average) with selectable time ranges: day, week, and month.
- **FR-019**: The temperature chart MUST clearly display the unit of measurement and allow the user to switch between Celsius and Fahrenheit display.
- **FR-020**: The temperature chart component MUST handle loading, empty, and error states explicitly — no blank or silent failure states.

### Key Entities

- **Temperature Record**: A single body temperature measurement captured for a user. Key attributes: user identifier, numeric value, unit (Celsius or Fahrenheit), ISO 8601 timestamp with timezone, source device identifier, ingestion source, optional measurement method, creation timestamp. Relationships: belongs to a user; associated with a device source.
- **Temperature Trend Summary**: An aggregated view of temperature records over a time period. Key attributes: user identifier, time range, granularity (day/week/month), minimum value, maximum value, average value, unit, device source filter. Derived from stored Temperature Records.
- **Health Metrics Catalogue Entry (Temperature)**: The registration of body temperature as a supported metric type within the platform. Key attributes: metric type identifier, display name, supported units, configurable validation range (min/max per unit).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user with a connected smart device can submit a body temperature reading and have it appear in their temperature chart within 30 seconds of submission under normal load conditions.
- **SC-002**: The platform correctly rejects 100% of temperature values outside the configured physiological range, returning a descriptive error message in each case.
- **SC-003**: Batch submissions of up to 100 temperature records are accepted and stored without data loss or duplication.
- **SC-004**: Temperature trend charts load and render completely within 3 seconds for any supported time range (day, week, month) under normal conditions.
- **SC-005**: The temperature chart displays accurate minimum, maximum, and average values that match the underlying stored records — 0% calculation error rate.
- **SC-006**: Unit switching between Celsius and Fahrenheit in the chart produces mathematically correct converted values with no rounding error exceeding 0.1 degrees.
- **SC-007**: Temperature data is present and accurate in analytics exports for any date range where temperature records exist.
- **SC-008**: The overall health dashboard loads without error when temperature data is present or absent.
- **SC-009**: Schema documentation is updated and available to integration partners before the feature is released.

## Assumptions

- The platform's existing physiological range defaults are: 34.0°C – 42.0°C (93.2°F – 107.6°F). These defaults are configurable via environment configuration and do not require a code change to update.
- The existing timeseries datastore (PostgreSQL with TimescaleDB) already supports the data volume expected from temperature metric ingestion; no infrastructure scaling is required as part of this story.
- The Kafka pipeline schema registry supports adding new Avro schema versions without disrupting existing consumers; the temperature schema will be a new schema, not a modification of an existing one.
- Device source identifiers are strings provided by the calling system; the platform stores them as-is and does not validate them against a device registry at ingestion time. Unrecognised device identifiers are flagged as "unverified" but accepted.
- The Celsius-to-Fahrenheit conversion formula used for display is the standard: °F = (°C × 9/5) + 32.
- The user's preferred display unit (Celsius or Fahrenheit) persists across sessions as part of their user profile or browser preference; defaulting to Celsius for new users is acceptable.
- The health analytics export endpoint already exists and supports pluggable metric types; adding temperature requires adding a new metric type handler, not a full endpoint rewrite.
- Measurement method (oral, axillary, tympanic) is optional metadata; absence of this field must not block ingestion or display.
- Batch idempotency is determined by the combination of user identifier + device source identifier + timestamp + value. Hash-based deduplication at ingestion time is the assumed mechanism.
- The recommendation engine and wellness summary service do not need to be updated as part of this story; temperature data will be available to them in a future story.
