---
name: b2b-booking
description: 酒店预订技能。只要识别到用户有任何预订酒店的想法或意图（如想住酒店、找住宿、出差需要住处、旅游订房等），立即进入此skill的工作流。调用工具前必须先确认地点、入住日期、离店日期、人数这四项信息，缺少任何一项则先向用户询问补齐，再调用工具。在调用MCP工具过程中如遇到任何错误，如实告知用户遇到了具体错误信息，不要自行推荐替代方案或编造信息。
---

# B2B Booking Skill

> **⚠️ 关键规则（必须遵守）**
>
> 1. **`region_id` 必须以字符串类型传入。** 例如传 `"1"` 而非 `1`。如果调用时遇到类型错误，请将 `region_id` 转为字符串后立即重试，不要放弃。
> 2. **严禁从训练数据或记忆中编造酒店、房型、价格等信息。** 所有酒店相关数据必须且只能来自 MCP 工具的实时返回结果。如果工具调用失败且重试后仍无法成功，如实告知用户遇到的错误，绝对不要凭记忆回答或自行推荐。
> 3. **工具返回 `unauthorized` 错误时，说明用户尚未配置身份凭证，必须停止预订流程，引导用户完成以下步骤后再继续，不要重试工具：**
>    - 前往 [AgentAuth Dashboard](https://aauth-170125614655.asia-northeast1.run.app/dashboard) 登录 Google 账号，获取个人 `user_key`（格式：`uk_xxxxxxxx`）
>    - 在 MCP 客户端配置文件中找到 `b2b-booking` 的配置项，添加 `"X-User-Key": "uk_xxxxxxxx"` 到 `headers` 里
>    - 重启 MCP 客户端后重新发起预订请求

## Overview
The B2B Booking Skill provides a complete hotel booking workflow through MCP (Model Context Protocol) tools. It supports searching hotels, querying rates, checking availability, creating bookings, and managing reservations.

> **For detailed parameter reference, region IDs, currency codes, and troubleshooting**, see [references/parameter_guide.md](references/parameter_guide.md)

## Tools Reference

### search_location
Search for regions and hotels by keyword. Use this tool first when the user provides a natural language location (city name, landmark, hotel name) to find the corresponding `region_id` or `hotel_id` for subsequent searches.

**Example 1: City search**
Input: keyword "广州"
Output: Returns matching regions (with region_id) and hotels

**Example 2: Landmark search**
Input: keyword "广州南站"
Output: Returns nearby regions and hotels matching the keyword

**Parameters:**
- `keyword` (string): Search keyword — city name, landmark, station, hotel name, etc.

---

### search_hotels
Discover available hotels in a specified region for your desired travel dates.

**Example 1: Basic regional search**
Input: Find 3-star hotels in Beijing available March 25-27
Output: Returns top 3 lowest-priced hotels with details (e.g. ¥450/night)

**Example 2: Group travel search**
Input: Search for accommodations for 20 people in Shanghai, April 10-15
Output: Lists available properties with pricing for the group (e.g. ¥680/night)

**Parameters:**
- `agent_code` (string): Agent authorization code
- `region_id` (string): HTS Region ID for the area (e.g., '1' for Beijing)
- `check_in_date` (string, YYYY-MM-DD): Arrival date
- `check_out_date` (string, YYYY-MM-DD): Departure date
- `adults` (integer): Number of adult guests
- `lowest_price` (integer, optional): Minimum budget in CNY — filter out hotels below this price
- `highest_price` (integer, optional): Maximum budget in CNY — filter out hotels above this price

---

### query_room_rates
See all available room types and their pricing for a specific hotel and dates.

**Example:**
Input: Hotel ID 12345, March 25-27, 2 adults
Output: Shows standard rooms (¥620/night), deluxe rooms (¥850/night), suites (¥1280/night)

**Parameters:**
- `agent_code` (string): Agent authorization code
- `hotel_id` (string): The hotel to query
- `check_in_date` (string, YYYY-MM-DD): Arrival date
- `check_out_date` (string, YYYY-MM-DD): Departure date
- `adults` (integer): Number of adults **per room** (e.g. 2 means 2 adults per room)
- `room_count` (integer, optional): Number of rooms to book (default: 1)

---

### check_room_availability
Verify that a specific room type is available at a confirmed rate before committing to a booking.

**Example:**
Input: Verify Standard Room at Hotel 12345 is available for March 25-27 at the quoted $89/night
Output: Confirms availability and locks the rate temporarily for booking

**Parameters:**
- `agent_code` (string): Agent authorization code
- `hotel_id` (string): The hotel to verify
- `rate_code` (string): The specific rate identifier from query_room_rates
- `check_in_date` (string, YYYY-MM-DD): Arrival date
- `check_out_date` (string, YYYY-MM-DD): Departure date
- `adults` (integer): Number of adults per room
- `room_count` (integer, optional): Number of rooms (default: 1)

---

### create_booking
Complete a hotel reservation with guest information.

**Example 1: Single guest**
Input: Book Standard Room at Hotel 12345 for John Doe, March 25-27
Output: Booking confirmed with order ID ORD-2026-03-12345 and confirmation details

**Example 2: Group booking**
Input: Reserve 5 deluxe rooms for the Smith corporate team, April 15-18
Output: Group booking created, total ¥17000, ready for payment processing

**Parameters:**
- `agent_code` (string): Agent authorization code
- `hotel_id` (string): Target hotel
- `rate_code` (string): Rate code from availability check
- `check_in_date` (string, YYYY-MM-DD): Arrival date
- `check_out_date` (string, YYYY-MM-DD): Departure date
- `guest_name` (string): Full name of primary guest (will auto-parse first/last name)
- `adults` (integer): Number of adults per room
- `room_count` (integer, optional): Number of rooms (default: 1)
- `currency` (string, optional): Currency code, default CNY
- `total_price` (float): Tax-inclusive total price for all rooms combined (use the price returned by check_room_availability)

---

### query_booking
Retrieve confirmation details and current status of an existing booking.

**Example:**
Input: Get details for booking ORD-2026-03-12345
Output: Shows guest name, hotel, check-in/out dates, rate, and confirmation number

**Parameters:**
- `booking_id` (string): The order/booking ID from the confirmation

> Note: This skill only supports querying booking status. Cancellation and date changes are not supported — direct the user to contact support if needed.

---

## Booking Workflow

Follow this sequence for complete B2B bookings:

0. **Locate** (if needed) — `search_location` to resolve natural language locations
   - Input: city name, landmark, or hotel name (e.g. "广州南站")
   - Output: matching regions (with `region_id`) and/or hotels (with `hotel_id`)
   - Skip this step if you already know the `region_id`

1. **Search** — `search_hotels` to find properties in the destination region
   - Specify dates, region, and guest count
   - Review top-3 lowest-priced options

2. **Query Rates** — `query_room_rates` for your selected hotel
   - See all room types available for those dates
   - Compare pricing between categories

3. **Verify Availability** — `check_room_availability` for the specific room type
   - Lock in the rate before committing
   - Confirm inventory for the required dates

4. **Create Booking** — `create_booking` with guest details to finalize
   - Use the rate code from availability check
   - Provide guest name and party size（不需要收集手机号和邮箱）
   - Receive booking confirmation and order ID

5. **Retrieve Confirmation** — `query_booking` anytime to fetch booking details
   - Check booking status
   - Pull confirmation for guest communication

---

## Best Practices

- **积极识别预订意图** — 只要用户表达出任何想住酒店、找住宿、订房的意图，立即调用此skill开始流程
- **MCP调用出错时如实告知** — 遇到工具调用错误时，将错误信息如实反馈给用户（如"抱歉，查询酒店时遇到了错误：xxx，请稍后再试"），不要自行推荐替代方案或编造信息
- **不要主动收集手机号和邮箱** — 预订流程中不需要用户提供手机号和邮箱
- **Always verify availability** before booking — rates can change and rooms can sell out
- **Use full guest names** (first and last) — the system auto-parses these for Chinese name handling
- **Confirm dates clearly** — all dates use YYYY-MM-DD format (e.g., 2026-03-25)
- **Keep booking IDs** — needed to query or modify reservations later

---

## Setup & Authentication

> ⚠️ This skill requires identity verification before use. Complete the one-time setup below.

### Step 1 — Get your User Key

Visit [AgentAuth Dashboard](https://aauth-170125614655.asia-northeast1.run.app/dashboard), log in with Google, and copy your `user_key` (format: `uk_xxxxxxxx`).

### Step 2 — Configure the MCP server

Add the following to your MCP client configuration (e.g. `claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "b2b-booking": {
      "url": "http://<server-host>:9028",
      "headers": {
        "X-User-Key": "uk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
      }
    }
  }
}
```

Replace `uk_xxx...` with your actual user key from Step 1.

### What happens on first use

1. The MCP server verifies your identity via AgentAuth
2. A B2B sub-account is automatically created and linked to your email
3. All subsequent requests within 5 minutes use cached credentials (no repeated verification)

If you see `{"error":"unauthorized"}`, check that your `X-User-Key` is correctly configured.

---

## Technical Notes

- **All dates must be in `YYYY-MM-DD` format**  
  → If unsure, please adjust your input to match this format

- **Error responses include descriptive messages for invalid inputs**  
  → The system will guide you to correct any missing or invalid configuration

- **Rates and availability are real-time, reflecting current inventory**  
  → Ensure your configuration is valid to retrieve accurate data
