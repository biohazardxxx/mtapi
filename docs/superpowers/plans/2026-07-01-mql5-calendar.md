# MT5 Economic Calendar Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Surface the native MQL5 economic calendar API (10 `Calendar*` functions) end-to-end through the MtApi5 connector.

**Architecture:** Bottom-up. Add C# enums → DTOs → command ids → client methods, then the MQL5 EA serializers → handlers → recompile `.ex5`, then a manual test-client hook. Each C# task is gated by `dotnet build`; MQL tasks are gated by a MetaEditor CLI compile with 0 errors.

**Tech Stack:** C# / .NET 8 (`MtApi5`), Newtonsoft.Json, MQL5 (`mq5/MtApi5.mq5` + `json.mqh`), MetaEditor64 CLI.

## Global Constraints

- Target framework: .NET 8.0; language/library patterns must match existing `MtApi5` code.
- JSON keys = native MQL5 field names (snake_case), matching `MqlRates`/`MqlTick` convention. Newtonsoft maps property-name → key by default.
- 64-bit fields (ids, `change_id`, datetimes, scaled `*_value`) must never round-trip through 32-bit `getInt`/`(int)`. Use `getLong` / `JSONNumber((long)...)`.
- Scaled `*_value` fields are integers ×1,000,000; `LONG_MIN` (`-9223372036854775808`) = "no value".
- New command ids: 307–316 (highest existing = `GetSymbols = 306`). Do not renumber existing commands.
- MQL recompile uses portable terminal `E:\MT5s\IC_Markets\MetaEditor64.exe`.
- Spec of record: `docs/superpowers/specs/2026-07-01-mql5-calendar-design.md`.
- Commit after every task. Do not push (user pushes).

## File Structure

New:
- `MtApi5/MqlCalendarCountry.cs` — country DTO
- `MtApi5/MqlCalendarEvent.cs` — event DTO
- `MtApi5/MqlCalendarValue.cs` — value DTO + `double?` accessors + internal `CalendarValueLastResult`

Modified:
- `MtApi5/Mt5Enums.cs` — +8 calendar enums
- `MtApi5/MtProtocol/Mt5CommandType.cs` — +10 command ids
- `MtApi5/MtApi5Client.cs` — +10 public methods
- `mq5/MtApi5.mq5` — +`GET_LONG_JSON_VALUE` macro, +3 serializers, +10 handlers, +10 `ADD_EXECUTOR`
- `mq5/MtApi5.ex5` — recompiled
- `TestClients/MtApi5TestClient/ViewModel.cs`, `MainWindow.xaml` — manual test hook

---

### Task 1: Calendar enums

**Files:**
- Modify: `MtApi5/Mt5Enums.cs` (append 8 enums)

**Interfaces:**
- Produces: `ENUM_CALENDAR_EVENT_TYPE`, `ENUM_CALENDAR_EVENT_SECTOR`, `ENUM_CALENDAR_EVENT_FREQUENCY`, `ENUM_CALENDAR_EVENT_TIMEMODE`, `ENUM_CALENDAR_EVENT_UNIT`, `ENUM_CALENDAR_EVENT_IMPORTANCE`, `ENUM_CALENDAR_EVENT_MULTIPLIER`, `ENUM_CALENDAR_EVENT_IMPACT` — consumed by Tasks 2 & 4.

- [ ] **Step 1: Append enums to `Mt5Enums.cs`** (values transcribed from MQL5 reference; each member = native constant value)

```csharp
public enum ENUM_CALENDAR_EVENT_TYPE
{
    CALENDAR_TYPE_EVENT = 0,
    CALENDAR_TYPE_INDICATOR = 1,
    CALENDAR_TYPE_HOLIDAY = 2
}

public enum ENUM_CALENDAR_EVENT_SECTOR
{
    CALENDAR_SECTOR_NONE = 0,
    CALENDAR_SECTOR_MARKET = 1,
    CALENDAR_SECTOR_GDP = 2,
    CALENDAR_SECTOR_JOBS = 3,
    CALENDAR_SECTOR_PRICES = 4,
    CALENDAR_SECTOR_MONEY = 5,
    CALENDAR_SECTOR_TRADE = 6,
    CALENDAR_SECTOR_GOVERNMENT = 7,
    CALENDAR_SECTOR_BUSINESS = 8,
    CALENDAR_SECTOR_CONSUMER = 9,
    CALENDAR_SECTOR_HOUSING = 10,
    CALENDAR_SECTOR_TAXES = 11,
    CALENDAR_SECTOR_HOLIDAYS = 12
}

public enum ENUM_CALENDAR_EVENT_FREQUENCY
{
    CALENDAR_FREQUENCY_NONE = 0,
    CALENDAR_FREQUENCY_WEEK = 1,
    CALENDAR_FREQUENCY_MONTH = 2,
    CALENDAR_FREQUENCY_QUARTER = 3,
    CALENDAR_FREQUENCY_YEAR = 4,
    CALENDAR_FREQUENCY_DAY = 5
}

public enum ENUM_CALENDAR_EVENT_TIMEMODE
{
    CALENDAR_TIMEMODE_DATETIME = 0,
    CALENDAR_TIMEMODE_DATE = 1,
    CALENDAR_TIMEMODE_NOTIME = 2,
    CALENDAR_TIMEMODE_TENTATIVE = 3
}

public enum ENUM_CALENDAR_EVENT_UNIT
{
    CALENDAR_UNIT_NONE = 0,
    CALENDAR_UNIT_PERCENT = 1,
    CALENDAR_UNIT_CURRENCY = 2,
    CALENDAR_UNIT_HOUR = 3,
    CALENDAR_UNIT_JOB = 4,
    CALENDAR_UNIT_RIG = 5,
    CALENDAR_UNIT_USD = 6,
    CALENDAR_UNIT_PEOPLE = 7,
    CALENDAR_UNIT_MORTGAGE = 8,
    CALENDAR_UNIT_VOTE = 9,
    CALENDAR_UNIT_BARREL = 10,
    CALENDAR_UNIT_CUBICFEET = 11,
    CALENDAR_UNIT_POSITION = 12,
    CALENDAR_UNIT_BUILDING = 13
}

public enum ENUM_CALENDAR_EVENT_IMPORTANCE
{
    CALENDAR_IMPORTANCE_NONE = 0,
    CALENDAR_IMPORTANCE_LOW = 1,
    CALENDAR_IMPORTANCE_MODERATE = 2,
    CALENDAR_IMPORTANCE_HIGH = 3
}

public enum ENUM_CALENDAR_EVENT_MULTIPLIER
{
    CALENDAR_MULTIPLIER_NONE = 0,
    CALENDAR_MULTIPLIER_THOUSANDS = 1,
    CALENDAR_MULTIPLIER_MILLIONS = 2,
    CALENDAR_MULTIPLIER_BILLIONS = 3,
    CALENDAR_MULTIPLIER_TRILLIONS = 4
}

public enum ENUM_CALENDAR_EVENT_IMPACT
{
    CALENDAR_IMPACT_NA = 0,
    CALENDAR_IMPACT_POSITIVE = 1,
    CALENDAR_IMPACT_NEGATIVE = 2
}
```

- [ ] **Step 2: Verify enum values against the platform.** Open `E:\MT5s\IC_Markets\MetaEditor64.exe` help (F1 → "Economic Calendar") or the MQL5 reference and confirm each numeric value above. These are stable language constants; correct any that differ (the C# integer MUST equal the native enum integer, because the EA sends `(int)nativeEnum`). Record confirmation in the commit message.

- [ ] **Step 3: Build to verify it compiles**

Run: `dotnet build MtApi5/MtApi5.csproj -c Release`
Expected: Build succeeded, 0 errors.

- [ ] **Step 4: Commit**

```bash
git add MtApi5/Mt5Enums.cs
git commit -m "MtApi5: add ENUM_CALENDAR_* calendar enums"
```

---

### Task 2: Calendar DTOs

**Files:**
- Create: `MtApi5/MqlCalendarCountry.cs`, `MtApi5/MqlCalendarEvent.cs`, `MtApi5/MqlCalendarValue.cs`

**Interfaces:**
- Consumes: enums from Task 1; `Mt5TimeConverter.ConvertFromMtTime(long)` (existing).
- Produces: `MqlCalendarCountry`, `MqlCalendarEvent`, `MqlCalendarValue`, internal `CalendarValueLastResult` — consumed by Task 4.

- [ ] **Step 1: Create `MtApi5/MqlCalendarCountry.cs`**

```csharp
namespace MtApi5
{
    public class MqlCalendarCountry
    {
        public long id { get; set; }
        public string name { get; set; }
        public string code { get; set; }
        public string currency { get; set; }
        public string currency_symbol { get; set; }
        public string url_name { get; set; }
    }
}
```

- [ ] **Step 2: Create `MtApi5/MqlCalendarEvent.cs`**

```csharp
namespace MtApi5
{
    public class MqlCalendarEvent
    {
        public long id { get; set; }
        public ENUM_CALENDAR_EVENT_TYPE type { get; set; }
        public ENUM_CALENDAR_EVENT_SECTOR sector { get; set; }
        public ENUM_CALENDAR_EVENT_FREQUENCY frequency { get; set; }
        public ENUM_CALENDAR_EVENT_TIMEMODE time_mode { get; set; }
        public long country_id { get; set; }
        public ENUM_CALENDAR_EVENT_UNIT unit { get; set; }
        public ENUM_CALENDAR_EVENT_IMPORTANCE importance { get; set; }
        public ENUM_CALENDAR_EVENT_MULTIPLIER multiplier { get; set; }
        public int digits { get; set; }
        public string source_url { get; set; }
        public string event_code { get; set; }
        public string name { get; set; }
    }
}
```

- [ ] **Step 3: Create `MtApi5/MqlCalendarValue.cs`** (includes the internal Last-result DTO)

```csharp
using System;
using System.Collections.Generic;

namespace MtApi5
{
    public class MqlCalendarValue
    {
        private const long Empty = long.MinValue; // LONG_MIN sentinel

        public long id { get; set; }
        public long event_id { get; set; }
        public long mt_time { get; set; }    // datetime (MT seconds)
        public long mt_period { get; set; }  // datetime (MT seconds)
        public int revision { get; set; }
        public long actual_value { get; set; }
        public long prev_value { get; set; }
        public long revised_prev_value { get; set; }
        public long forecast_value { get; set; }
        public ENUM_CALENDAR_EVENT_IMPACT impact_type { get; set; }

        public DateTime time => Mt5TimeConverter.ConvertFromMtTime(mt_time);
        public DateTime period => Mt5TimeConverter.ConvertFromMtTime(mt_period);
        public double? ActualValue      => actual_value       == Empty ? (double?)null : actual_value       / 1e6;
        public double? PrevValue        => prev_value         == Empty ? (double?)null : prev_value         / 1e6;
        public double? RevisedPrevValue => revised_prev_value == Empty ? (double?)null : revised_prev_value / 1e6;
        public double? ForecastValue    => forecast_value     == Empty ? (double?)null : forecast_value     / 1e6;
    }

    internal class CalendarValueLastResult
    {
        public long ChangeId { get; set; }
        public List<MqlCalendarValue> Values { get; set; }
    }
}
```

- [ ] **Step 4: Build to verify it compiles**

Run: `dotnet build MtApi5/MtApi5.csproj -c Release`
Expected: Build succeeded, 0 errors.

- [ ] **Step 5: Commit**

```bash
git add MtApi5/MqlCalendarCountry.cs MtApi5/MqlCalendarEvent.cs MtApi5/MqlCalendarValue.cs
git commit -m "MtApi5: add calendar DTOs (country, event, value)"
```

---

### Task 3: Command ids

**Files:**
- Modify: `MtApi5/MtProtocol/Mt5CommandType.cs` (add 10 members after `GetSymbols = 306`)

**Interfaces:**
- Produces: `Mt5CommandType.CalendarCountries` (307) … `CalendarValueLast` (316) — consumed by Task 4 and mirrored in Task 6.

- [ ] **Step 1: Add the 10 members**

```csharp
CalendarCountries = 307,
CalendarCountryById = 308,
CalendarEventById = 309,
CalendarEventByCountry = 310,
CalendarEventByCurrency = 311,
CalendarValueById = 312,
CalendarValueHistoryByEvent = 313,
CalendarValueHistory = 314,
CalendarValueLastByEvent = 315,
CalendarValueLast = 316,
```

- [ ] **Step 2: Build**

Run: `dotnet build MtApi5/MtApi5.csproj -c Release`
Expected: Build succeeded, 0 errors.

- [ ] **Step 3: Commit**

```bash
git add MtApi5/MtProtocol/Mt5CommandType.cs
git commit -m "MtApi5: add calendar command ids 307-316"
```

---

### Task 4: C# client methods

**Files:**
- Modify: `MtApi5/MtApi5Client.cs` (add 10 public methods in the market-info region, near `CopyRates`)

**Interfaces:**
- Consumes: Tasks 1–3; existing `SendCommand<T>(ExecutorHandle, Mt5CommandType, object)`, `Mt5TimeConverter.ConvertToMtTime(DateTime)`.
- Produces: the 10 public calendar methods (final API surface).

- [ ] **Step 1: Add the methods**

```csharp
public MqlCalendarCountry[] CalendarCountries()
{
    // Empty object "{}" (not null): the EA's GET_JSON_PAYLOAD needs a valid JSON object.
    var p = new Dictionary<string, object>();
    var r = SendCommand<List<MqlCalendarCountry>>(ExecutorHandle, Mt5CommandType.CalendarCountries, p);
    return r?.ToArray() ?? [];
}

public bool CalendarCountryById(long countryId, out MqlCalendarCountry country)
{
    var p = new Dictionary<string, object> { { "CountryId", countryId } };
    country = SendCommand<MqlCalendarCountry>(ExecutorHandle, Mt5CommandType.CalendarCountryById, p);
    return country != null;
}

public bool CalendarEventById(long eventId, out MqlCalendarEvent evt)
{
    var p = new Dictionary<string, object> { { "EventId", eventId } };
    evt = SendCommand<MqlCalendarEvent>(ExecutorHandle, Mt5CommandType.CalendarEventById, p);
    return evt != null;
}

public MqlCalendarEvent[] CalendarEventByCountry(string countryCode)
{
    var p = new Dictionary<string, object> { { "CountryCode", countryCode ?? string.Empty } };
    var r = SendCommand<List<MqlCalendarEvent>>(ExecutorHandle, Mt5CommandType.CalendarEventByCountry, p);
    return r?.ToArray() ?? [];
}

public MqlCalendarEvent[] CalendarEventByCurrency(string currency)
{
    var p = new Dictionary<string, object> { { "Currency", currency ?? string.Empty } };
    var r = SendCommand<List<MqlCalendarEvent>>(ExecutorHandle, Mt5CommandType.CalendarEventByCurrency, p);
    return r?.ToArray() ?? [];
}

public bool CalendarValueById(long valueId, out MqlCalendarValue value)
{
    var p = new Dictionary<string, object> { { "ValueId", valueId } };
    value = SendCommand<MqlCalendarValue>(ExecutorHandle, Mt5CommandType.CalendarValueById, p);
    return value != null;
}

public MqlCalendarValue[] CalendarValueHistoryByEvent(long eventId, DateTime from, DateTime to = default)
{
    var p = new Dictionary<string, object>
    {
        { "EventId", eventId },
        { "FromDate", Mt5TimeConverter.ConvertToMtTime(from) },
        { "ToDate", to == default ? 0 : Mt5TimeConverter.ConvertToMtTime(to) }
    };
    var r = SendCommand<List<MqlCalendarValue>>(ExecutorHandle, Mt5CommandType.CalendarValueHistoryByEvent, p);
    return r?.ToArray() ?? [];
}

public MqlCalendarValue[] CalendarValueHistory(DateTime from, DateTime to = default,
                                               string countryCode = null, string currency = null)
{
    var p = new Dictionary<string, object>
    {
        { "FromDate", Mt5TimeConverter.ConvertToMtTime(from) },
        { "ToDate", to == default ? 0 : Mt5TimeConverter.ConvertToMtTime(to) },
        { "CountryCode", countryCode ?? string.Empty },
        { "Currency", currency ?? string.Empty }
    };
    var r = SendCommand<List<MqlCalendarValue>>(ExecutorHandle, Mt5CommandType.CalendarValueHistory, p);
    return r?.ToArray() ?? [];
}

public MqlCalendarValue[] CalendarValueLastByEvent(long eventId, ref long changeId)
{
    var p = new Dictionary<string, object> { { "EventId", eventId }, { "ChangeId", changeId } };
    var r = SendCommand<CalendarValueLastResult>(ExecutorHandle, Mt5CommandType.CalendarValueLastByEvent, p);
    changeId = r?.ChangeId ?? changeId;
    return r?.Values?.ToArray() ?? [];
}

public MqlCalendarValue[] CalendarValueLast(ref long changeId, string countryCode = null, string currency = null)
{
    var p = new Dictionary<string, object>
    {
        { "ChangeId", changeId },
        { "CountryCode", countryCode ?? string.Empty },
        { "Currency", currency ?? string.Empty }
    };
    var r = SendCommand<CalendarValueLastResult>(ExecutorHandle, Mt5CommandType.CalendarValueLast, p);
    changeId = r?.ChangeId ?? changeId;
    return r?.Values?.ToArray() ?? [];
}
```

- [ ] **Step 2: Confirm the `SendCommand<T>` overload used.** Verify `MtApi5Client.cs` exposes `SendCommand<T>(int expertHandle, Mt5CommandType, object payload)` returning `T?` (the CopyRates path). If the in-repo signature differs (e.g. requires a client arg), match the exact form used by `CopyRates`. Adjust the calls above to the real signature.

- [ ] **Step 3: Build**

Run: `dotnet build MtApi5/MtApi5.csproj -c Release`
Expected: Build succeeded, 0 errors.

- [ ] **Step 4: Commit**

```bash
git add MtApi5/MtApi5Client.cs
git commit -m "MtApi5: add 10 Calendar* client methods"
```

---

### Task 5: MQL5 — `GET_LONG_JSON_VALUE` macro + 3 serializers

**Files:**
- Modify: `mq5/MtApi5.mq5` (add macro near other `GET_*_JSON_VALUE`; add serializers near `MqlRatesToJson`)

**Interfaces:**
- Consumes: `json.mqh` (`getLong`, `JSONNumber(long)`, `JSONArray`, `JSONObject`, `JSONString`).
- Produces: macro `GET_LONG_JSON_VALUE`; functions `MqlCalendarCountryToJson`, `MqlCalendarEventToJson`, `MqlCalendarValueToJson` — consumed by Task 6.

- [ ] **Step 1: Add the macro** (next to the existing `GET_INT_JSON_VALUE` definition)

```mql5
#define GET_LONG_JSON_VALUE(json, name_value, return_value) \
   CHECK_JSON_VALUE(json, name_value); long return_value = json.p.getLong(name_value)
```

- [ ] **Step 2: Add the 3 serializers** (near `MqlRatesToJson`)

```mql5
JSONObject* MqlCalendarCountryToJson(const MqlCalendarCountry& c)
{
   JSONObject *jo = new JSONObject();
   jo.put("id", new JSONNumber((long)c.id));
   jo.put("name", new JSONString(c.name));
   jo.put("code", new JSONString(c.code));
   jo.put("currency", new JSONString(c.currency));
   jo.put("currency_symbol", new JSONString(c.currency_symbol));
   jo.put("url_name", new JSONString(c.url_name));
   return jo;
}

JSONObject* MqlCalendarEventToJson(const MqlCalendarEvent& e)
{
   JSONObject *jo = new JSONObject();
   jo.put("id", new JSONNumber((long)e.id));
   jo.put("type", new JSONNumber((int)e.type));
   jo.put("sector", new JSONNumber((int)e.sector));
   jo.put("frequency", new JSONNumber((int)e.frequency));
   jo.put("time_mode", new JSONNumber((int)e.time_mode));
   jo.put("country_id", new JSONNumber((long)e.country_id));
   jo.put("unit", new JSONNumber((int)e.unit));
   jo.put("importance", new JSONNumber((int)e.importance));
   jo.put("multiplier", new JSONNumber((int)e.multiplier));
   jo.put("digits", new JSONNumber((int)e.digits));
   jo.put("source_url", new JSONString(e.source_url));
   jo.put("event_code", new JSONString(e.event_code));
   jo.put("name", new JSONString(e.name));
   return jo;
}

JSONObject* MqlCalendarValueToJson(const MqlCalendarValue& v)
{
   JSONObject *jo = new JSONObject();
   jo.put("id", new JSONNumber((long)v.id));
   jo.put("event_id", new JSONNumber((long)v.event_id));
   jo.put("mt_time", new JSONNumber((long)v.time));
   jo.put("mt_period", new JSONNumber((long)v.period));
   jo.put("revision", new JSONNumber((int)v.revision));
   jo.put("actual_value", new JSONNumber((long)v.actual_value));
   jo.put("prev_value", new JSONNumber((long)v.prev_value));
   jo.put("revised_prev_value", new JSONNumber((long)v.revised_prev_value));
   jo.put("forecast_value", new JSONNumber((long)v.forecast_value));
   jo.put("impact_type", new JSONNumber((int)v.impact_type));
   return jo;
}
```

- [ ] **Step 3: Confirm `JSONNumber((long))` prints full 64-bit precision.** Inspect `mq5/json.mqh` `JSONNumber(long)` / `getLong` to confirm it stores/serializes a 64-bit long (not truncated to int). If `toString()` formats via a double or `IntegerToString((int))`, note it — 64-bit ids need `LongToString`-style output. (Existing `getLong` usage implies support; verify before relying on it.)

- [ ] **Step 4: Commit** (compiles as part of Task 6; commit source now for a clean history)

```bash
git add mq5/MtApi5.mq5
git commit -m "MQL5: add GET_LONG_JSON_VALUE macro and calendar serializers"
```

---

### Task 6: MQL5 — handlers + executor registration

**Files:**
- Modify: `mq5/MtApi5.mq5` (add 10 `Execute_Calendar*` handlers near other executors; add 10 `ADD_EXECUTOR` lines in the registration block)

**Interfaces:**
- Consumes: serializers + macro from Task 5; native `Calendar*` functions; `GET_STRING_JSON_VALUE`, `CreateSuccessResponse`, `CreateErrorResponse`, `GET_JSON_PAYLOAD`.
- Produces: dispatchable commands 307–316.

- [ ] **Step 1: Add `ADD_EXECUTOR` registrations** (with the other `ADD_EXECUTOR` calls)

```mql5
ADD_EXECUTOR(307, CalendarCountries);
ADD_EXECUTOR(308, CalendarCountryById);
ADD_EXECUTOR(309, CalendarEventById);
ADD_EXECUTOR(310, CalendarEventByCountry);
ADD_EXECUTOR(311, CalendarEventByCurrency);
ADD_EXECUTOR(312, CalendarValueById);
ADD_EXECUTOR(313, CalendarValueHistoryByEvent);
ADD_EXECUTOR(314, CalendarValueHistory);
ADD_EXECUTOR(315, CalendarValueLastByEvent);
ADD_EXECUTOR(316, CalendarValueLast);
```

- [ ] **Step 2: Add array-getter handlers** (307, 310, 311)

```mql5
string Execute_CalendarCountries()
{
   MqlCalendarCountry countries[];
   int count = CalendarCountries(countries);
   if(count < 0) return CreateErrorResponse(GetLastError(), "CalendarCountries failed");
   JSONArray* ja = new JSONArray();
   for(int i = 0; i < count; i++) ja.put(i, MqlCalendarCountryToJson(countries[i]));
   return CreateSuccessResponse(ja);
}

string Execute_CalendarEventByCountry()
{
   GET_JSON_PAYLOAD(jo);
   GET_STRING_JSON_VALUE(jo, "CountryCode", country_code);
   MqlCalendarEvent events[];
   int count = CalendarEventByCountry(country_code, events);
   if(count < 0) return CreateErrorResponse(GetLastError(), "CalendarEventByCountry failed");
   JSONArray* ja = new JSONArray();
   for(int i = 0; i < count; i++) ja.put(i, MqlCalendarEventToJson(events[i]));
   return CreateSuccessResponse(ja);
}

string Execute_CalendarEventByCurrency()
{
   GET_JSON_PAYLOAD(jo);
   GET_STRING_JSON_VALUE(jo, "Currency", currency);
   MqlCalendarEvent events[];
   int count = CalendarEventByCurrency(currency, events);
   if(count < 0) return CreateErrorResponse(GetLastError(), "CalendarEventByCurrency failed");
   JSONArray* ja = new JSONArray();
   for(int i = 0; i < count; i++) ja.put(i, MqlCalendarEventToJson(events[i]));
   return CreateSuccessResponse(ja);
}
```

- [ ] **Step 3: Add single-getter handlers** (308, 309, 312) — not-found returns success with null Value → C# `false`

```mql5
string Execute_CalendarCountryById()
{
   GET_JSON_PAYLOAD(jo);
   GET_LONG_JSON_VALUE(jo, "CountryId", country_id);
   MqlCalendarCountry country;
   ResetLastError();
   if(CalendarCountryById(country_id, country))
      return CreateSuccessResponse(MqlCalendarCountryToJson(country));
   int err = GetLastError();
   if(err == 0) return CreateSuccessResponse(NULL);           // not found
   return CreateErrorResponse(err, "CalendarCountryById failed");
}

string Execute_CalendarEventById()
{
   GET_JSON_PAYLOAD(jo);
   GET_LONG_JSON_VALUE(jo, "EventId", event_id);
   MqlCalendarEvent event;
   ResetLastError();
   if(CalendarEventById((ulong)event_id, event))
      return CreateSuccessResponse(MqlCalendarEventToJson(event));
   int err = GetLastError();
   if(err == 0) return CreateSuccessResponse(NULL);
   return CreateErrorResponse(err, "CalendarEventById failed");
}

string Execute_CalendarValueById()
{
   GET_JSON_PAYLOAD(jo);
   GET_LONG_JSON_VALUE(jo, "ValueId", value_id);
   MqlCalendarValue value;
   ResetLastError();
   if(CalendarValueById((ulong)value_id, value))
      return CreateSuccessResponse(MqlCalendarValueToJson(value));
   int err = GetLastError();
   if(err == 0) return CreateSuccessResponse(NULL);
   return CreateErrorResponse(err, "CalendarValueById failed");
}
```

- [ ] **Step 4: Add value-history handlers** (313, 314)

```mql5
string Execute_CalendarValueHistoryByEvent()
{
   GET_JSON_PAYLOAD(jo);
   GET_LONG_JSON_VALUE(jo, "EventId", event_id);
   GET_LONG_JSON_VALUE(jo, "FromDate", from_date);
   GET_LONG_JSON_VALUE(jo, "ToDate", to_date);
   MqlCalendarValue values[];
   int count = CalendarValueHistoryByEvent((ulong)event_id, values, (datetime)from_date, (datetime)to_date);
   if(count < 0) return CreateErrorResponse(GetLastError(), "CalendarValueHistoryByEvent failed");
   JSONArray* ja = new JSONArray();
   for(int i = 0; i < count; i++) ja.put(i, MqlCalendarValueToJson(values[i]));
   return CreateSuccessResponse(ja);
}

string Execute_CalendarValueHistory()
{
   GET_JSON_PAYLOAD(jo);
   GET_LONG_JSON_VALUE(jo, "FromDate", from_date);
   GET_LONG_JSON_VALUE(jo, "ToDate", to_date);
   GET_STRING_JSON_VALUE(jo, "CountryCode", country_code);
   GET_STRING_JSON_VALUE(jo, "Currency", currency);
   MqlCalendarValue values[];
   int count = CalendarValueHistory(values, (datetime)from_date, (datetime)to_date, country_code, currency);
   if(count < 0) return CreateErrorResponse(GetLastError(), "CalendarValueHistory failed");
   JSONArray* ja = new JSONArray();
   for(int i = 0; i < count; i++) ja.put(i, MqlCalendarValueToJson(values[i]));
   return CreateSuccessResponse(ja);
}
```

- [ ] **Step 5: Add last-value handlers** (315, 316) — response body carries the updated `change_id`

```mql5
string Execute_CalendarValueLastByEvent()
{
   GET_JSON_PAYLOAD(jo);
   GET_LONG_JSON_VALUE(jo, "EventId", event_id);
   GET_LONG_JSON_VALUE(jo, "ChangeId", change_id_in);
   ulong change_id = (ulong)change_id_in;
   MqlCalendarValue values[];
   int count = CalendarValueLastByEvent((ulong)event_id, change_id, values);
   if(count < 0) return CreateErrorResponse(GetLastError(), "CalendarValueLastByEvent failed");
   JSONArray* ja = new JSONArray();
   for(int i = 0; i < count; i++) ja.put(i, MqlCalendarValueToJson(values[i]));
   JSONObject* body = new JSONObject();
   body.put("ChangeId", new JSONNumber((long)change_id));
   body.put("Values", ja);
   return CreateSuccessResponse(body);
}

string Execute_CalendarValueLast()
{
   GET_JSON_PAYLOAD(jo);
   GET_LONG_JSON_VALUE(jo, "ChangeId", change_id_in);
   GET_STRING_JSON_VALUE(jo, "CountryCode", country_code);
   GET_STRING_JSON_VALUE(jo, "Currency", currency);
   ulong change_id = (ulong)change_id_in;
   MqlCalendarValue values[];
   int count = CalendarValueLast(change_id, values, country_code, currency);
   if(count < 0) return CreateErrorResponse(GetLastError(), "CalendarValueLast failed");
   JSONArray* ja = new JSONArray();
   for(int i = 0; i < count; i++) ja.put(i, MqlCalendarValueToJson(values[i]));
   JSONObject* body = new JSONObject();
   body.put("ChangeId", new JSONNumber((long)change_id));
   body.put("Values", ja);
   return CreateSuccessResponse(body);
}
```

- [ ] **Step 6: Empty-string → NULL for optional args.** MQL treats `""` and `NULL` differently for `country_code`/`currency`. Verify the native functions accept `""` as "no filter"; if not, add `if(StringLen(country_code)==0) country_code=NULL;` before the calls in `Execute_CalendarValueHistory` and `Execute_CalendarValueLast`. Confirm against runtime in Task 9.

- [ ] **Step 7: Confirm `CreateSuccessResponse(NULL)` yields absent/null `Value`.** Read `CreateSuccessResponse` in `mq5/MtApi5.mq5`: when passed `NULL` it must produce `{"ErrorCode":"0"}` (no `Value`, or `Value:null`) so `SendCommand<T>` returns `default(T)`. If it dereferences the arg unconditionally, guard it or return a hand-built `{"ErrorCode":"0"}` string in the three single-getters.

- [ ] **Step 8: Commit**

```bash
git add mq5/MtApi5.mq5
git commit -m "MQL5: add calendar command handlers (307-316)"
```

---

### Task 7: Recompile `MtApi5.ex5`

**Files:**
- Modify: `mq5/MtApi5.ex5` (regenerated binary — also clears the stale binary from the upstream sync)

**Interfaces:** none (build artifact).

- [ ] **Step 1: Stage includes into the portable terminal**

```bash
cp mq5/hash.mqh mq5/json.mqh "E:/MT5s/IC_Markets/MQL5/Include/"
cp mq5/MtApi5.mq5 "E:/MT5s/IC_Markets/MQL5/Experts/MtApi5.mq5"
```

- [ ] **Step 2: Compile via MetaEditor CLI**

Run (PowerShell):
```
& "E:\MT5s\IC_Markets\MetaEditor64.exe" /compile:"E:\MT5s\IC_Markets\MQL5\Experts\MtApi5.mq5" /include:"E:\MT5s\IC_Markets\MQL5" /log:"E:\MT5s\IC_Markets\MQL5\Experts\MtApi5_compile.log"
```
Note: MetaEditor returns immediately; wait for the process to exit, then read the log.

- [ ] **Step 3: Verify the compile log**

Read `E:\MT5s\IC_Markets\MQL5\Experts\MtApi5_compile.log`.
Expected: `0 errors`. Review any warnings; new calendar code must add none. If errors → fix in Task 5/6 source and recompile (do not edit the `.ex5`).

- [ ] **Step 4: Copy the binary back**

```bash
cp "E:/MT5s/IC_Markets/MQL5/Experts/MtApi5.ex5" mq5/MtApi5.ex5
```

- [ ] **Step 5: Commit**

```bash
git add mq5/MtApi5.ex5
git commit -m "MQL5: recompile MtApi5.ex5 with calendar support"
```

---

### Task 8: Test-client hook

**Files:**
- Modify: `TestClients/MtApi5TestClient/ViewModel.cs`, `TestClients/MtApi5TestClient/MainWindow.xaml`

**Interfaces:**
- Consumes: `MtApi5Client.CalendarValueHistory` (Task 4).
- Produces: a manual "Calendar" UI action.

- [ ] **Step 1: Read the existing patterns.** Open `ViewModel.cs` and find an existing parameterless command (e.g. a `RelayCommand`/`ICommand` wired to a button that calls the client and appends to an output/log collection). Mirror its exact idiom — command field, backing method, and how results are surfaced.

- [ ] **Step 2: Add the command method to `ViewModel.cs`** (adapt the log call to the existing logging mechanism found in Step 1)

```csharp
private void ExecuteCalendarValueHistory()
{
    try
    {
        var from = DateTime.Now.AddDays(-7);
        var values = _mtApiClient.CalendarValueHistory(from);
        Log($"CalendarValueHistory: {values.Length} values since {from:u}");
        foreach (var v in values)
            Log($"  event={v.event_id} time={v.time:u} actual={v.ActualValue} " +
                $"forecast={v.ForecastValue} prev={v.PrevValue} impact={v.impact_type}");
    }
    catch (Exception ex)
    {
        Log($"CalendarValueHistory error: {ex.Message}");
    }
}
```

- [ ] **Step 3: Wire the command + button.** Add the `ICommand` property in `ViewModel.cs` following the existing pattern, and a `<Button Content="Calendar History" Command="{Binding CalendarHistoryCommand}"/>` in `MainWindow.xaml` in the same panel as the other test buttons.

- [ ] **Step 4: Build the test client**

Run: `dotnet build TestClients/MtApi5TestClient/MtApi5TestClient.csproj -c Release`
Expected: Build succeeded, 0 errors. (If the project is x64-only, add `-p:Platform=x64`.)

- [ ] **Step 5: Commit**

```bash
git add TestClients/MtApi5TestClient/ViewModel.cs TestClients/MtApi5TestClient/MainWindow.xaml
git commit -m "MtApi5TestClient: add manual calendar history test action"
```

---

### Task 9: Manual runtime verification (live terminal)

**Files:** none (verification only).

**Interfaces:** exercises the full stack against a running MT5 terminal with the recompiled EA attached.

- [ ] **Step 1: Attach & connect.** Load the recompiled `MtApi5.ex5` on a chart in `E:\MT5s\IC_Markets` (DLL imports allowed, calendar populated). Start `MtApi5TestClient`, connect to the MT5 port.

- [ ] **Step 2: Array path.** Click "Calendar History". Expected: non-empty list for the last 7 days; `actual`/`forecast`/`prev` decode to sane numbers; empty fields show as `null` (not a huge negative).

- [ ] **Step 3: Single-getter path.** From code/console, call `CalendarEventById(knownId, out e)` → `true` with populated fields; `CalendarEventById(0, out e)` → `false`, no exception.

- [ ] **Step 4: Change-id path.** Call `CalendarValueLast(ref changeId)` twice; confirm `changeId` is updated (non-zero) and the second call with the returned id behaves per the native semantics (delta only).

- [ ] **Step 5: 64-bit integrity.** Confirm a large `event_id` / `value_id` round-trips exactly (no truncation) between C# and MQL.

- [ ] **Step 6: Record results** in the PR description / a short note. If any decode is wrong, fix the offending enum value (Task 1) or serializer (Task 5) and recompile (Task 7).

---

## Notes on TDD adaptation

This codebase has no automated test project; the network+terminal boundary can't be unit-tested in isolation here. Compile gates (`dotnet build`, MetaEditor log) are the fast feedback loop; Task 9 is the real behavioral gate and requires the live terminal. Keep commits per-task so a bad enum value or serializer bug is bisectable.
