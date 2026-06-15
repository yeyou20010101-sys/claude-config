# 完善 backend/tests 测试覆盖

## Context

当前 `backend/tests` 有 19 个测试文件、124 个用例，但覆盖深度不均衡——7 个文件已有充分覆盖，12 个文件仅测了 basic happy path，边界/异常路径几乎空白。本计划在不改动源码的前提下，新增约 **46 个测试**，按优先级分三批推进。

## 不改动的文件（已充分覆盖）

`test_auth_validators` `test_cloud_asr_config` `test_skill_loader` `test_phase3_skills` `test_creative_session_timeline` `test_intent_router_references` `test_platform_config`

---

## Phase 1 — 关键缺口（23 tests）

### 1.1 `test_user_balance.py` → 1→8 tests

**被测**: `services/user_balance.py`

- 抽公共 fixture `balance_db`（`tmp_path` + `monkeypatch` `AUTH_DATABASE_PATH`）
- 新增：`test_deduct_zero_amount` / `test_deduct_negative_amount` / `test_deduct_invalid_user` (parametrize 0, -1) / `test_get_balance_invalid_user` / `test_set_balance_clamps_negative` / `test_deduct_floors_at_zero` / `test_ensure_column_idempotent`

### 1.2 `test_media_errors.py` → 1→8 tests

**被测**: `services/media_errors.py`（纯函数，无需 fixture）

- 新增：`test_quota_error_humanized` / `test_balance_error_humanized` / `test_rate_limit_humanized` / `test_long_error_truncated` / `test_non_video_task_error` / `test_format_failure_non_video_skill` / `test_humanize_none_error`

### 1.3 `test_model_call_log.py` → 1→6 tests

**被测**: `services/model_call_log.py`

- 用 `tmp_path` 隔离 `_DB_PATH`
- 新增：`test_context_propagation` / `test_clear_context_resets` / `test_record_error_truncation` / `test_list_recent_default_limit` / `test_list_recent_clamped_limit`

### 1.4 `test_usage_records.py` → 1→5 tests

**被测**: `services/usage_records.py`

- 新增：`test_empty_usage_returns_zero_summary` / `test_page_beyond_range` / `test_time_range_today_vs_week` / `test_agent_filter_all_vs_specific`

---

## Phase 2 — 显著缺口（13 tests）

### 2.1 `test_video_media_modes.py` → 2→6 tests

**被测**: `film-agent/llm_client.py` `_build_dashscope_video_media`

- 新增：`test_single_first_frame_mode` / `test_continue_missing_clip_url` / `test_first_last_missing_url` / `test_text_only_mode`

### 2.2 `test_film_assistant_reply.py` → 3→7 tests

**被测**: `film-agent/llm_client.py` `canned_assistant_reply` / `try_assistant_reply`

- 新增：`test_canned_generation_failed` / `test_canned_generation_submitted_video` / `test_canned_generation_completed` / `test_canned_data_with_error`

### 2.3 `test_model_manager.py` → 2→5 tests

**被测**: `services/model_manager.py`

- 用 `monkeypatch` 重定向 `_CONFIG_PATH` 到 `tmp_path` 避免污染生产配置
- 新增：`test_set_and_get_active` / `test_get_all_active_returns_dict` / `test_unknown_agent_falls_back_to_film`

### 2.4 `test_dashboard_stats.py` → 2→4 tests

**被测**: `services/dashboard_stats.py`

- 新增：`test_empty_stats_returns_zeros` / `test_stats_different_time_ranges`

---

## Phase 3 — 中等提升（10 tests）

### 3.1 `test_media_token_cost.py` → 3→6 tests

- 新增：`test_non_media_type_unchanged` / `test_existing_usage_wins` / `test_none_usage_error_status`

### 3.2 `test_reference_images.py` → 4→7 tests

- 新增：`test_truncation_above_three`（需构造 5 张不同的小 PNG）/ `test_empty_list_entries_filtered` / `test_both_params_none`

### 3.3 `test_image_refine_and_ark.py` → 6→8 tests

- 新增：`test_empty_message_not_refine` / `test_regenerate_intent_is_refine`

### 3.4 `test_skills_manager.py` → 4→6 tests

- 新增：`test_path_traversal_with_backslashes` / `test_delete_nonexistent_skill`

---

## 关键风险与对策

| 风险 | 对策 |
|---|---|
| `model_manager._CONFIG_PATH` 指向生产文件 | `monkeypatch.setattr(mm, "_CONFIG_PATH", tmp_path)` 在首次调用前注入 |
| `model_call_log._DB_PATH` 全局变量跨测试污染 | 每个测试用独立 `tmp_path` |
| 5 张不同 PNG 构造复杂 | 生成 1x1 像素不同颜色的 PNG，或 mock `normalize_image_base64` |
| 仪表盘测试涉及时区 (Asia/Shanghai) | 复用现有 `test_dashboard_stats_aggregates` 的时区处理方式 |

## 验证

```bash
cd d:/02_Work/xiaojingyu
python -m pytest backend/tests/ -v
# 预期：~170 passed（原 124 + 新增 46）
```
