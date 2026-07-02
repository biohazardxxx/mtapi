# MtApi5 — MT5 Economic Calendar Support (Design)

Date: 2026-07-01
Branch: `feat/mql5-calendar` (off synced `dev`)
Status: Approved design (pending spec review)

## 1. Goal

MT5/MQL5 exposes the terminal's built-in economic calendar natively
(`CalendarValueHistory`, `CalendarEventById`, `CalendarValueLast`, and the rest of
the `Calendar*` family), but none of it is surfaced through the MtApi5 connector.

This feature adds **full parity** — all 10 native calendar functions — end-to-end:
C# client (`MtApi5`) → command enum → MQL5 Expert Advisor (`mq5/MtApi5.mq5`) → back.

Non-goal (YAGNI): no caching, no polling/subscription of calendar changes, no
event-stream push. Callers poll via `CalendarValueLast*` and its `change_id` exactly
as the native API intends.

## 2. Native API being wrapped

MQL5 signatures (for reference; the wrappers mirror these):

```
int  CalendarCountries(MqlCalendarCountry &countries[]);
bool CalendarCountryById(const long country_id, MqlCalendarCountry &country);
bool CalendarEventById(const ulong event_id, MqlCalendarEvent &event);
int  CalendarEventByCountry(const string country_code, MqlCalendarEvent &events[]);
int  CalendarEventByCurrency(const string currency, MqlCalendarEvent &events[]);
bool CalendarValueById(const ulong value_id, MqlCalendarValue &value);
int  CalendarValueHistoryByEvent(const ulong event_id, MqlCalendarValue &values[],
                                 const datetime datetime_from, const datetime datetime_to=0);
int  CalendarValueHistory(MqlCalendarValue &values[], const datetime datetime_from,
                          const datetime datetime_to=0,
                          const string country_code=NULL, const string currency=NULL);
int  CalendarValueLastByEvent(const ulong event_id, ulong &change_id, MqlCalendarValue &values[]);
int  CalendarValueLast(ulong &change_id, MqlCalendarValue &values[],
                       const string country_code=NULL, const string currency=NULL);
```

### Structs

```
MqlCalendarCountry { ulong id; string name; string code; string currency;
                     string currency_symbol; string url_name; }

MqlCalendarEvent   { ulong id; ENUM_CALENDAR_EVENT_TYPE type;
                     ENUM_CALENDAR_EVENT_SECTOR sector;
                     ENUM_CALENDAR_EVENT_FREQUENCY frequency;
                     ENUM_CALENDAR_EVENT_TIMEMODE time_mode; ulong country_id;
                     ENUM_CALENDAR_EVENT_UNIT unit;
                     ENUM_CALENDAR_EVENT_IMPORTANCE importance;
                     ENUM_CALENDAR_EVENT_MULTIPLIER multiplier; uint digits;
                     string source_url; string event_code; string name; }

MqlCalendarValue   { ulong id; ulong event_id; datetime time; datetime period;
                     int revision; long actual_value; long prev_value;
                     long revised_prev_value; long forecast_value;
                     ENUM_CALENDAR_EVENT_IMPACT impact_type; }
```

`*_value` fields are integers scaled ×1,000,000; the sentinel `LONG_MIN`
(`-9223372036854775808`) means "no value".

## 3. Command numbers

Highest existing command is `GetSymbols = 306`. New commands take 307–316
(`MtApi5/MtProtocol/Mt5CommandType.cs`):

| Cmd | Name |
|-----|------|
| 307 | CalendarCountries |
| 308 | CalendarCountryById |
| 309 | CalendarEventById |
| 310 | CalendarEventByCountry |
| 311 | CalendarEventByCurrency |
| 312 | CalendarValueById |
| 313 | CalendarValueHistoryByEvent |
| 314 | CalendarValueHistory |
| 315 | CalendarValueLastByEvent |
| 316 | CalendarValueLast |

## 4. C# enums — added to `MtApi5/Mt5Enums.cs`

Eight enums, integer values mirroring MQL5 exactly:

- `ENUM_CALENDAR_EVENT_TYPE` — EVENT=0, INDICATOR=1, HOLIDAY=2
- `ENUM_CALENDAR_EVENT_SECTOR` — NONE=0, MARKET, GDP, JOBS, PRICES, MONEY, TRADE,
  GOVERNMENT, BUSINESS, CONSUMER, HOUSING, TAXES, HOLIDAYS (values per MQL5 docs)
- `ENUM_CALENDAR_EVENT_FREQUENCY` — NONE=0, WEEK, MONTH, QUARTER, YEAR, DAY
- `ENUM_CALENDAR_EVENT_TIMEMODE` — DATETIME=0, DATE, NOTIME, TENTATIVE
- `ENUM_CALENDAR_EVENT_UNIT` — NONE=0, PERCENT, CURRENCY, HOUR, JOB, RIG, USD,
  PEOPLE, MORTGAGE, VOTE, BARREL, CUBICFEET, POSITION, BUILDING (values per MQL5 docs)
- `ENUM_CALENDAR_EVENT_IMPORTANCE` — NONE=0, LOW, MODERATE, HIGH
- `ENUM_CALENDAR_EVENT_MULTIPLIER` — NONE=0, THOUSANDS, MILLIONS, BILLIONS, TRILLIONS
- `ENUM_CALENDAR_EVENT_IMPACT` — NA=0, POSITIVE, NEGATIVE

Exact numeric values will be transcribed from the MetaEditor `Calendar` enum
definitions during implementation and cross-checked against a live terminal.

## 5. C# DTOs — 3 new files in `MtApi5/`

JSON keys use the native MQL5 field names (snake_case), matching the existing
`MqlRates`/`MqlTick` convention. Newtonsoft maps property name → JSON key by default.

### `MqlCalendarCountry.cs`
```csharp
public class MqlCalendarCountry
{
    public long id { get; set; }
    public string name { get; set; }
    public string code { get; set; }
    public string currency { get; set; }
    public string currency_symbol { get; set; }
    public string url_name { get; set; }
}
```

### `MqlCalendarEvent.cs`
```csharp
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
```

### `MqlCalendarValue.cs`
```csharp
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

    // Convenience accessors (not serialized)
    public DateTime time   => Mt5TimeConverter.ConvertFromMtTime(mt_time);
    public DateTime period => Mt5TimeConverter.ConvertFromMtTime(mt_period);
    public double? ActualValue       => actual_value       == Empty ? (double?)null : actual_value       / 1e6;
    public double? PrevValue         => prev_value         == Empty ? (double?)null : prev_value         / 1e6;
    public double? RevisedPrevValue  => revised_prev_value == Empty ? (double?)null : revised_prev_value / 1e6;
    public double? ForecastValue     => forecast_value     == Empty ? (double?)null : forecast_value     / 1e6;
}
```

## 6. C# client methods — `MtApi5/MtApi5Client.cs`

Follows the existing `CopyRates` pattern: build a `Dictionary<string,object>` payload,
call `SendCommand<T>`, convert result.

```csharp
public MqlCalendarCountry[] CalendarCountries()
{
    // Send an empty object "{}" (not null): the EA's GET_JSON_PAYLOAD requires a
    // valid JSON object even though this command reads no parameters.
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

public bool CalendarEventById(long eventId, out MqlCalendarEvent evt) { /* Value=null => false */ }
public bool CalendarValueById(long valueId, out MqlCalendarValue value) { /* Value=null => false */ }

public MqlCalendarEvent[] CalendarEventByCountry(string countryCode) { /* payload {CountryCode} */ }
public MqlCalendarEvent[] CalendarEventByCurrency(string currency)   { /* payload {Currency} */ }

public MqlCalendarValue[] CalendarValueHistoryByEvent(long eventId, DateTime from, DateTime to = default)
{
    var p = new Dictionary<string, object> {
        { "EventId", eventId },
        { "FromDate", Mt5TimeConverter.ConvertToMtTime(from) },
        { "ToDate", to == default ? 0 : Mt5TimeConverter.ConvertToMtTime(to) } };
    var r = SendCommand<List<MqlCalendarValue>>(ExecutorHandle, Mt5CommandType.CalendarValueHistoryByEvent, p);
    return r?.ToArray() ?? [];
}

public MqlCalendarValue[] CalendarValueHistory(DateTime from, DateTime to = default,
                                               string countryCode = null, string currency = null)
{
    var p = new Dictionary<string, object> {
        { "FromDate", Mt5TimeConverter.ConvertToMtTime(from) },
        { "ToDate", to == default ? 0 : Mt5TimeConverter.ConvertToMtTime(to) },
        { "CountryCode", countryCode ?? string.Empty },
        { "Currency", currency ?? string.Empty } };
    var r = SendCommand<List<MqlCalendarValue>>(ExecutorHandle, Mt5CommandType.CalendarValueHistory, p);
    return r?.ToArray() ?? [];
}

// change_id is in/out. EA returns { ChangeId, Values:[...] }.
public MqlCalendarValue[] CalendarValueLastByEvent(long eventId, ref long changeId)
{
    var p = new Dictionary<string, object> { { "EventId", eventId }, { "ChangeId", changeId } };
    var r = SendCommand<CalendarValueLastResult>(ExecutorHandle, Mt5CommandType.CalendarValueLastByEvent, p);
    changeId = r?.ChangeId ?? changeId;
    return r?.Values?.ToArray() ?? [];
}

public MqlCalendarValue[] CalendarValueLast(ref long changeId, string countryCode = null, string currency = null)
{
    var p = new Dictionary<string, object> {
        { "ChangeId", changeId },
        { "CountryCode", countryCode ?? string.Empty },
        { "Currency", currency ?? string.Empty } };
    var r = SendCommand<CalendarValueLastResult>(ExecutorHandle, Mt5CommandType.CalendarValueLast, p);
    changeId = r?.ChangeId ?? changeId;
    return r?.Values?.ToArray() ?? [];
}
```

Internal helper DTO (in `MtApi5/MtProtocol/` or alongside the value DTO):
```csharp
internal class CalendarValueLastResult
{
    public long ChangeId { get; set; }
    public List<MqlCalendarValue> Values { get; set; }
}
```

**`bool` + `out` semantics** for single getters (308/309/312): the EA returns
`ErrorCode = 0` with `Value = null` whenever the native call returns `false`, so
`SendCommand<T>` returns `default(T)` (`null`) without throwing — the wrapper maps
`null → false`.

> **Verified-behavior note (updated after runtime testing):** the design originally
> planned to still throw for "genuine" failures by inspecting `GetLastError()`. In
> practice the native `CalendarCountryById/EventById/ValueById` set a nonzero
> `LastError` even for a simple not-found (e.g. `id = 0`), so there is no reliable way
> to separate "not found" from "error". The handlers therefore map **any** native
> `false` to `Value = null` (client `bool = false`), faithfully mirroring the native
> `bool` contract. Callers that need an error code can read it out-of-band; the
> connector surface is the `bool`.

Empty string args (`CountryCode`/`Currency` = "") are treated as `NULL` on the MQL side.

## 7. MQL5 EA — `mq5/MtApi5.mq5` + `mq5/json.mqh`

### 7.1 New macro (`MtApi5.mq5`, near the other `GET_*_JSON_VALUE` macros)
```
#define GET_LONG_JSON_VALUE(json, name_value, return_value) \
   CHECK_JSON_VALUE(json, name_value); long return_value = json.p.getLong(name_value)
```
Used for 64-bit params: ids, `change_id`, and datetimes (`FromDate`/`ToDate`),
which exceed 32-bit `getInt`.

### 7.2 Executor registration
```
ADD_EXECUTOR(307, CalendarCountries);
ADD_EXECUTOR(308, CalendarCountryById);
... through ...
ADD_EXECUTOR(316, CalendarValueLast);
```

### 7.3 Serializers (mirror `MqlRatesToJson`)
```
JSONObject* MqlCalendarCountryToJson(const MqlCalendarCountry& c);
JSONObject* MqlCalendarEventToJson(const MqlCalendarEvent& e);
JSONObject* MqlCalendarValueToJson(const MqlCalendarValue& v);
```
- All ids / `country_id` / `event_id` / scaled `*_value` fields → `new JSONNumber((long)...)`.
- `time` / `period` → `new JSONNumber((long)v.time)` under keys `mt_time` / `mt_period`.
- Enum fields → `new JSONNumber((int)e.type)` etc.

### 7.4 Handlers (`Execute_Calendar*`)
- **Array getters** (307, 310, 311, 313, 314): call native fn into a local
  `MqlCalendarXxx array[]`, loop `jaresult.put(i, XxxToJson(array[i]))`,
  `return CreateSuccessResponse(jaresult)`. On a native return of `-1`,
  `return CreateErrorResponse(GetLastError(), "...")`.
- **Single getters** (308, 309, 312): call native `bool` fn.
  If `true` → `CreateSuccessResponse(XxxToJson(item))`.
  If `false` and `GetLastError()==0` (simply not found) →
  `CreateSuccessResponse(NULL)` (Value omitted/null → C# `false`).
  If `false` with a real error → `CreateErrorResponse(GetLastError(), "...")`.
- **Last getters** (315, 316): read `ChangeId` (in), call native fn with a `ulong change_id`
  seeded from it, build `JSONObject{ "ChangeId": (long)change_id, "Values": jaValues }`,
  `return CreateSuccessResponse(thatObject)`.

`datetime_to == 0` is passed straight through to the native call (open-ended range),
which is why C# sends `0` for a default `to`.

## 8. Data flow / wire examples

`CalendarValueHistory(from, to, "US", null)` request payload:
```json
{ "FromDate": 1719792000, "ToDate": 0, "CountryCode": "US", "Currency": "" }
```
Success response:
```json
{ "ErrorCode": "0", "Value": [
  { "id": 173300, "event_id": 840030016, "mt_time": 1719792000, "mt_period": 1717200000,
    "revision": 0, "actual_value": 3200000, "prev_value": 3000000,
    "revised_prev_value": -9223372036854775808, "forecast_value": 3100000, "impact_type": 1 }
] }
```
`CalendarValueLast` response body carries the round-tripped change id:
```json
{ "ErrorCode": "0", "Value": { "ChangeId": 123456789, "Values": [ ... ] } }
```

## 9. Test client hook — `TestClients/MtApi5TestClient`

Add a "Calendar" action (button in `MainWindow.xaml` + command in `ViewModel.cs`)
that calls `CalendarValueHistory(DateTime.Now.AddDays(-7))` and prints each returned
value (event id, time, actual/forecast/prev) to the client's output log. Manual
verification only — requires a connected MT5 terminal with the calendar populated.

## 10. Build / compile

C# (quick iteration): `dotnet build MtApi5/MtApi5.csproj -c Release`.

MQL5 recompile (this repo does not build `.ex5` via dotnet):
1. Copy `mq5/hash.mqh` and `mq5/json.mqh` into `E:\MT5s\IC_Markets\MQL5\Include\`.
2. Copy `mq5/MtApi5.mq5` into `E:\MT5s\IC_Markets\MQL5\Experts\` (or compile in place with `/include`).
3. Compile:
   ```
   E:\MT5s\IC_Markets\MetaEditor64.exe /compile:"<path>\MtApi5.mq5" ^
       /include:"E:\MT5s\IC_Markets\MQL5" /log:"compile.log"
   ```
4. Copy the produced `MtApi5.ex5` back to `mq5/MtApi5.ex5`, verify the compile log is
   error-free (warnings reviewed), and commit.

Note: the pending upstream-sync `.ex5` (source has merged upstream changes, binary not
yet regenerated) is resolved by this same recompile.

## 11. Verification

- **Build:** `MtApi5.csproj` compiles clean (Release).
- **MQL compile:** MetaEditor log shows 0 errors for `MtApi5.mq5`.
- **Runtime (manual, live terminal):** the test-client Calendar action returns a non-empty
  set for a broad recent range; single getters resolve a known event id (`true`) and
  return `false` for a bogus id without throwing; `CalendarValueLast` updates `changeId`.

## 12. Files touched

New:
- `MtApi5/MqlCalendarCountry.cs`
- `MtApi5/MqlCalendarEvent.cs`
- `MtApi5/MqlCalendarValue.cs`
- (`CalendarValueLastResult` — internal, colocated)

Modified:
- `MtApi5/MtProtocol/Mt5CommandType.cs` (+10 enum members)
- `MtApi5/Mt5Enums.cs` (+8 enums)
- `MtApi5/MtApi5Client.cs` (+10 public methods)
- `mq5/MtApi5.mq5` (+macro, +10 executors, +10 handlers, +3 serializers)
- `mq5/MtApi5.ex5` (recompiled)
- `TestClients/MtApi5TestClient/MainWindow.xaml`, `ViewModel.cs` (test hook)
