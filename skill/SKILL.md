---
name: b2b-booking
description: 酒店预订技能。**仅当**用户明确表达要预订酒店、查询房价、或确认已有订单时才触发（例如"帮我订酒店"、"查一下北京的酒店"、"我要预定房间"）。纯粹的旅游计划、问路、景点推荐等不触发此skill。进入工作流后，调用工具前必须先确认地点、入住日期、离店日期、人数这四项信息，缺少任何一项则先向用户询问补齐。在调用MCP工具过程中如遇到任何错误，如实告知用户遇到了具体错误信息，不要自行推荐替代方案或编造信息。
metadata.openclaw: {"emoji": "🏨", "primaryEnv": "user_key.txt", "mcpServer": "http://39.108.114.224:9061"}
---

# B2B Booking Skill

> **⚠️ 关键规则（必须遵守）**
>
> 1. **`region_id` 必须以字符串类型传入。** 例如传 `"1"` 而非 `1`。如果调用时遇到类型错误，请将 `region_id` 转为字符串后立即重试，不要放弃。
> 2. **严禁从训练数据或记忆中编造酒店、房型、价格等信息。** 所有酒店相关数据必须且只能来自 MCP 工具的实时返回结果。如果工具调用失败且重试后仍无法成功，如实告知用户遇到的错误，绝对不要凭记忆回答或自行推荐。
> 3. **工具返回 `unauthorized` 错误时，说明 user_key 无效或已过期，必须停止预订流程：删除 `{baseDir}/user_key.txt`，提示用户前往 [AgentAuth Dashboard](https://aauth-170125614655.asia-northeast1.run.app/dashboard) 重新获取 user_key 后再继续，不要重试工具。**

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
- `user_key` (string): Read from `{baseDir}/user_key.txt`
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
- `user_key` (string): Read from `{baseDir}/user_key.txt`
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
- `user_key` (string): Read from `{baseDir}/user_key.txt`
- `hotel_id` (string): The hotel to query
- `check_in_date` (string, YYYY-MM-DD): Arrival date
- `check_out_date` (string, YYYY-MM-DD): Departure date
- `adults` (integer): Number of adults **per room** (e.g. 2 means 2 adults per room)
- `room_count` (integer, optional): Number of rooms (default: 1)

---

### check_room_availability
Verify that a specific room type is available at a confirmed rate before committing to a booking.

**Example:**
Input: Verify Standard Room at Hotel 12345 is available for March 25-27 at the quoted $89/night
Output: Confirms availability and locks the rate temporarily for booking

**Parameters:**
- `user_key` (string): Read from `{baseDir}/user_key.txt`
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
- `user_key` (string): Read from `{baseDir}/user_key.txt`
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
- `user_key` (string): Read from `{baseDir}/user_key.txt`
- `agent_ref_id` (string): The order reference ID from create_booking confirmation

> Note: This skill only supports querying booking status. Cancellation and date changes are not supported — direct the user to contact support if needed.

---

### pay_order
Initiate payment for a confirmed booking via WeChat Pay or Alipay.

**After create_booking succeeds, always ask the user to choose a payment method before calling this tool.**

**Example:**
Input: Pay for booking TM20260416001 with WeChat Pay
Output: Returns a payment URL for the user to complete payment

**Parameters:**
- `user_key` (string): Read from `{baseDir}/user_key.txt`
- `agent_ref_id` (string): Order reference ID from create_booking
- `payment_type` (integer): `11` = WeChat Pay, `12` = Alipay
- `return_url` (string, optional): Redirect URL after payment completes

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

6. **Pay** — `pay_order` to initiate payment after booking is confirmed
   - Ask the user to choose: 微信支付 (payment_type=11) or 支付宝 (payment_type=12)
   - Call pay_order with the chosen method
   - Share the returned pay_url with the user to complete payment

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

## Setup

Before calling any tool, check the user key **first**.

### Step 1 — User Key

1. Read `{baseDir}/user_key.txt`
2. If the file is **missing or empty** — do NOT proceed to tool calls. Instead, tell the user:
   > "在开始之前，需要先验证你的身份。请前往 https://aauth-170125614655.asia-northeast1.run.app/dashboard 用 Google 账号登录，复制你的 `user_key`（格式：`uk_xxxxxxxx`），然后告诉我。"
   Once the user provides the key, save it to `{baseDir}/user_key.txt`, then proceed.
3. If the file **exists and has content** — use the stored value as `user_key` for all tool calls. Do NOT ask the user again.
4. If any tool returns `unauthorized` — the key is invalid or expired. Delete `{baseDir}/user_key.txt`, then repeat step 2.

---

## Technical Notes

- **All dates must be in `YYYY-MM-DD` format**  
  → If unsure, please adjust your input to match this format

- **Error responses include descriptive messages for invalid inputs**  
  → The system will guide you to correct any missing or invalid configuration

- **Rates and availability are real-time, reflecting current inventory**  
  → Ensure your configuration is valid to retrieve accurate data
