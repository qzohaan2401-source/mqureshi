# MySQL → PostgreSQL Migration: Table & Column Mapping

**Project:** AnnexCloud my2pg-migrator
**Last Updated:** 2026-04-23

---

## Table of Contents

1. [Global Notes](#global-notes)
2. [Sites & Config](#1-sites--config)
3. [Users](#2-users)
4. [Actions](#3-actions)
5. [Campaigns](#4-campaigns)
6. [Badges](#5-badges)
7. [Tiers](#6-tiers)
8. [Groups (Hierarchy)](#7-groups-hierarchy)
9. [Segments](#8-segments)
10. [Rewards](#9-rewards)
11. [Transactions & Points](#10-transactions--points)
12. [Stores](#11-stores)
13. [Programs & Timezones](#12-programs--timezones)
14. [Localisation](#13-localisation)
15. [Unmapped MySQL Tables](#unmapped-mysql-tables)
16. [PostgreSQL Tables with No MySQL Source](#postgresql-tables-with-no-mysql-source)
17. [Appendix: EAV → JSONB Flattening Pattern](#appendix-eav--jsonb-flattening-pattern)

---

## Global Notes

### multitemplate_id

> **Important:** `multitemplate_id` appears in nearly every MySQL source table but is **not** an unmapped column.
>
> In the new PostgreSQL schema, the multitemplate concept has been redesigned as a **sites hierarchy**:
> - `multitemplate_id` → `sites.parent_id`
> - A site now references its parent/template site via `parent_id` on the `sites` table.
> - `multitemplate_id` is resolved at the **sites level only** and is **not carried as a direct column** in any other migrated table.

### EAV Custom Attributes

Several MySQL tables store custom attributes using an Entity-Attribute-Value (EAV) pattern via paired `_attribute` and `_attribute_values` tables. During migration these are flattened into a `custom_attributes` **JSONB** column on the corresponding PostgreSQL table. See [Appendix](#appendix-eav--jsonb-flattening-pattern) for full details.

### Pre-assigned UUIDs (Migrate Verbatim)

The following MySQL tables had a **UUID column added during migration prep**. The UUID value must be copied exactly to the PostgreSQL `id` column — do **not** use `gen_random_uuid()` for these tables.

| MySQL Table | MySQL UUID Column | PostgreSQL Table |
|---|---|---|
| `s15_v3_action_group` | `id` | `action_groups` |
| `s15_v3_campaign_group` | `id` | `campaign_groups` |
| `s15_v3_hierarchy_groups` | `id` | `groups` |
| `s15_v3_reward_detail` | `id` | `rewards` |
| `s15_v3_segment` | `id` | `segments` |
| `s15_v3_tier` | `id` | `tiers` |
| `s15_v3_campaign_detail` | **`uuid`** *(column named `uuid`, not `id`)* | `campaigns` |

> **Migration rule:** SELECT the UUID column from MySQL and INSERT it as the PG `id`. Strip whitespace before use. Skip rows where the value is blank or NULL after stripping.

### Legend

| Notation | Meaning |
|---|---|
| `enum: {0:false, 1:true}` | MySQL integer/tinyint mapped to PostgreSQL boolean |
| `lookup UUID via <field>` | The PG column stores a UUID FK resolved from the legacy integer ID stored in `migrated_<entity>_id` |
| `auto-generated UUID` | PG `id` column is auto-populated via `gen_random_uuid()` — no MySQL source |
| `verbatim UUID` | PG `id` is copied directly from a pre-assigned UUID column in MySQL — see Pre-assigned UUIDs note above |
| `subset of columns` | Only a portion of the MySQL source table's columns map to this PG table |
| `(derived)` | Value is computed or inferred during migration, not read directly from MySQL |
| `— see global note` | Refers to the `multitemplate_id` note above |

---

## 1. Sites & Config

### Table Mapping Summary

| MySQL Table(s) | PostgreSQL Table | Notes |
|---|---|---|
| `site_detail` | `sites` | + ref: `s15_incentive_master` |
| `site_detail` + `s15_v3_loyalty_setting` + `s15_v3_configuration_detail` + `s15_v3_hierarchy_settings` | `loyalty_settings` | Multi-source join |
| `site_detail` + `s15_v3_configuration_detail` + `s15_v3_loyalty_setting` + `s15_v2_tab_setting` + `s15_v3_member_defination` | `feature_flags` | Multi-source join |
| `s15_v3_give_points_reason` | `site_point_reason_configs` | |
| Multi-source (10 attribute tables) | `app_attribute_settings` | One row per attribute `name` + `entity_type` |

---

### 1.1 `site_detail` → `sites`

**Sources:** `site_detail` (primary), `s15_incentive_master` (joined for `multitemplate_id` / `parent_id`)

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `id` | |
| `site_name` | `name` | |
| `timezone` | `timezone` | |
| `date_format` + `time_format` | `date_time_format` | Concatenated |
| `active` | `is_active` | `enum: {0:false, 1:true}` |
| `admin_email` | `created_by` | |
| `secret_key` | `secret_key` | |
| `db_add_date` | `created_at` | |
| `db_update_date` | `updated_at` | |
| `s15_incentive_master.multitemplate_id` | `multitemplate_id` | From joined ref table; default `0` |
| `s15_incentive_master.site_id` | `parent_id` | From joined ref table; default `NULL` |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL columns** (not migrated to `sites` — many feed `loyalty_settings` and `feature_flags`):
`user_id`, `user_type`, `display_branding`, `admin_person_name`, `admin_phone`, `admin_password`, `enable_buckets`, `loyalty_points_format`, `loyalty_program`, `loyalty_tier_sequence_flag`, `enable_loyalty_tier_period`, `use_python_backend`, `use_v2_config_backend`, and ~80 additional legacy UI/feature flag columns.

**Unmapped PostgreSQL columns:** None — all PG columns are populated.
</details>

---

### 1.2 `site_detail` + refs → `loyalty_settings`

**Sources:** `s15_v3_loyalty_setting` (primary), `site_detail`, `s15_v3_configuration_detail`, `s15_v3_hierarchy_settings` (all joined on `site_id`)

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `s15_v3_loyalty_setting.site_id` | `site_id` | |
| `s15_v3_configuration_detail.credits_to_currency_ratio` | `currency_per_credit` | |
| `site_detail.loyalty_points_format` | `points_rounding` | `enum: {0:ceil, 1:floor, 2:decimal, 3:round}` |
| `site_detail.loyalty_program` | `program_type` | `enum: {0:implicit, 1:explicit_with_retro, 2:explicit_without_retro, 3:opt_in_with_retro}` |
| `s15_v3_loyalty_setting.partial_return_redeem_refund` | `refund_on_partial_return_enabled` | `enum: {0:false, 1:true}` |
| `s15_v3_loyalty_setting.enable_disable_order_discount` | `exclude_discounts_from_points` | `enum: {0:false, 1:true}` |
| `s15_v3_loyalty_setting.enable_join_benefit_create_user` | `join_benefits_enabled` | `enum: {0:false, 1:true}` |
| `s15_v3_loyalty_setting.enable_remove_actionseries_benefit` | `allow_actionseries_benefit_removal` | `enum: {0:false, 1:true}` |
| `s15_v3_configuration_detail.multiple_campaign_benefit` | `multi_campaign_benefit_enabled` | `enum: {0:false, 1:true}` |
| `s15_v3_loyalty_setting.enable_campaign_group` | `campaign_group_enabled` | `enum: {0:false, 1:true}` |
| `s15_v3_configuration_detail.allow_negative_points` | `negative_points_policy` | `enum: {0:DISALLOW, 1:ALLOW, 2:SET_ZERO}` |
| `s15_v3_configuration_detail.enable_sitelevel_customid_limit` | `action_customid_enabled` | `enum: {0:false, 1:true}` |
| `s15_v3_configuration_detail.site_level_customId_limit` | `customid_action_limit` | |
| `s15_v3_hierarchy_settings.group_level_admin_rule` | `enable_group_level_setting` | `enum: {0:false, 1:true}` |
| `s15_v3_loyalty_setting.display_separate_campaign_milestone_benefit_activity` | `separate_campaign_milestone_benefit_activity` | `enum: {0:false, 1:true}` |
| `site_detail.loyalty_tier_sequence_flag` | `sequential_tier_enabled` | `enum: {0:false, 1:true}` |
| `site_detail.enable_loyalty_tier_period` | `tier_period_enabled` | `enum: {0:false, 1:true}` |
| `s15_v3_loyalty_setting.disable_tier_downgrade_in_order_return_or_cancel` | `tier_downgrade_policy` | `enum: {0:ALLOW, 1:BLOCK_ON_RETURN_OR_CANCEL}` |
| `s15_v3_configuration_detail.tier_downgrade_in_upgrade_period` | `tier_downgrade_during_upgrade_period` | `enum: {0:false, 1:true}` |
| `s15_v3_configuration_detail.upgrade_period` | `tier_upgrade_period_type` | `enum: {0:null, 1:CALENDAR, 2:ANNIVERSARY}` |
| `s15_v3_configuration_detail.upgrade_period_duration` | `tier_upgrade_period_years` | |
| `s15_v3_configuration_detail.maintain_period` | `tier_maintain_period_type` | `enum: {0:null, 1:CALENDAR, 2:ANNIVERSARY}` |
| `s15_v3_configuration_detail.maintain_period_duration` | `tier_maintain_period_years` | |
| `s15_v3_configuration_detail.enable_include_tier_limit` | `tier_action_limit_enabled` | `enum: {0:false, 1:true}` |
| `s15_v3_configuration_detail.auto_delivery_enable` | `auto_delivery_enabled` | `enum: {0:false, 1:true}` |
| *(no MySQL source)* | `order_discount_disabled` | No MySQL source; default `false` |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL columns:** `id`, `group_member_limit`, `member_revoke`, `hierarchy_status`, `terms_condition`, `enable_incentive_engine`, `enable_net_price`, `create_date`, `update_date`, and ~60 additional operational/feature flag columns not migrated to `loyalty_settings`.

**Unmapped PostgreSQL columns:** `id` (auto-generated UUID).
</details>

---

### 1.3 `site_detail` + refs → `feature_flags`

**Sources:** `site_detail` (primary), `s15_v3_configuration_detail`, `s15_v3_loyalty_setting`, `s15_v2_tab_setting`, `s15_v3_member_defination` (all joined on `site_id`)

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_detail.site_id` | `site_id` | |
| `site_detail.enable_buckets` | `multi_bucket_enabled` | `enum: {0:false, 1:true}` |
| `s15_v3_configuration_detail.tierv2_migration_status` | `is_tier_v2` | `enum: {0:false, 1:true}` |
| `s15_v3_loyalty_setting.enable_customize_orderdate_in_orderapi` | `backdated_order_allowed` | `enum: {0:false, 1:true}` |
| `s15_v3_loyalty_setting.hierarchy_status` | `is_hierarchy_active` | `enum: {0:false, 1:true}` |
| `s15_v2_tab_setting.multitemplate_active` | `multi_template_allowed` | `enum: {0:false, 1:true}` |
| `s15_v2_tab_setting.enable_receipt_submission` | `receipt_submission_enabled` | `enum: {0:false, 1:true}` |
| `s15_v3_member_defination.member_level` | `member_definition_rules_enabled` | `enum: {0:false, 1:true}` |
| `s15_v3_loyalty_setting.enable_survey_flag` | `survey_enabled` | `enum: {0:false, 1:true}` |
| `s15_v3_loyalty_setting.enable_multi_language_flag` | `multilanguage_enabled` | `enum: {0:false, 1:true}` |
| `s15_v3_loyalty_setting.is_webhook_active` | `webhook_enabled` | `enum: {0:false, 1:true}` |
| `site_detail.use_python_backend` | `use_v2_backend` | `enum: {0:false, 1:true}` |
| `site_detail.use_v2_config_backend` | `use_v2_config_backend` | `enum: {0:false, 1:true}` |

<details>
<summary>Unmapped columns</summary>

**Unmapped PostgreSQL columns:** `id` (auto-generated UUID).
</details>

---

### 1.4 `s15_v3_give_points_reason` → `site_point_reason_configs`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `reason` | `reason` | |
| `flag` | `type` | `enum: {0:null, 1:MODERATION, 2:POINTS, 3:REWARD}` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped PostgreSQL:** `id` (auto-generated UUID).
</details>

---

### 1.5 Multi-source → `app_attribute_settings`

**Sources:** Multiple MySQL attribute definition tables. Each source table contributes rows with a hardcoded `entity_type` value. One PG row is produced per `(site_id, name, entity_type)` combination.

| MySQL Source Table | PG `entity_type` Value | Notes |
|---|---|---|
| `s15_v3_order_attribute` | `order` | Has `data_type` column |
| `s15_v3_user_attribute` | `user` | Has `data_type` column |
| `s15_v3_badge_attributes` | `badge` | No `data_type`; defaults `VARCHAR` |
| `s15_v3_campaign_attributes` | `campaign` | No `data_type`; defaults `VARCHAR` |
| `s15_v3_journey_attributes` | `journey` | No `data_type`; defaults `VARCHAR` |
| `s15_v3_action_attribute` | `action` | Has `data_type` column |
| `s15_v3_issuance_order_attribute_master` | `order` | Merged with `s15_v3_order_attribute`; no `data_type` |
| `s15_v3_issuance_product_attribute_master` | `item` | No `data_type`; defaults `VARCHAR` |
| `s15_v3_reward_attribute` | `reward` | Has `data_type` column |
| `s15_v3_store_attribute` | `store` | No `data_type`; defaults `VARCHAR` |

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `attribute_name` | `name` | |
| *(hardcoded per source)* | `entity_type` | Set per source table — see table above |
| `data_type` | `data_type` | `VARCHAR` used as default for source tables without a `data_type` column |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |
| `visible_in_report` | `visible_in_report` | Only present in `s15_v3_store_attribute`, `s15_v3_reward_attribute`, `s15_v3_action_attribute`; `false` for all other sources |

<details>
<summary>Unmapped columns</summary>

**Unmapped PostgreSQL:** `id` (auto-generated UUID), `display_name` (no MySQL source; defaults `NULL`), `description` (no MySQL source), `default_value` (no MySQL source), `is_mandatory` (no MySQL source).
</details>

---

## 2. Users

### Table Mapping Summary

| MySQL Table(s) | PostgreSQL Table | Notes |
|---|---|---|
| `user` + `s15_v3_user_detail` | `users` | Combined source |
| `s15_v3_user_attribute` + `s15_v3_user_attribute_values` | `users` | `custom_attributes` JSONB column |
| `s15_v3_user_blacklist` | `blacklisted_users` | |
| `s15_v3_domain_blacklist` | `blacklisted_domains` | |
| `s15_v3_merge_user_detail` | `merge_user_requests` | |
| `s15_v3_user_rewards` | `reward_members` | |
| `s15_v3_user_inactive_status_history` | `member_statuses` | |

---

### 2.1 `user` + `s15_v3_user_detail` → `users`

**Sources:** `user` (primary), `s15_v3_user_detail` (joined on `user.user_id = s15_v3_user_detail.user_id`)

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `user.site_id` | `site_id` | |
| `user.uid` | `member_id` | |
| `user.email_address` | `email` | |
| `user.first_name` | `first_name` | |
| `user.last_name` | `last_name` | |
| `user.phone` | `phone` | |
| `user.city` | `city` | |
| `user.state` | `state` | |
| `user.zipcode` | `zipcode` | |
| `user.country` | `country` | |
| `user.birth_date` | `birth_date` | |
| `user.anniversary_date` | `anniversary_date` | |
| `user.user_profile_image_url` | `profile_image_url` | |
| `user.status` | `status` | `enum: {0:inactive, 1:active}` |
| `user.is_deleted` | `is_deleted` | `enum: {0:false, 1:true}` |
| `user.create_date` | `created_at` | |
| `user.update_date` | `updated_at` | |
| `user.country_id` | `country_id` | Fallback: `s15_v3_user_detail.country_id` if `user.country_id = 0` |
| `s15_v3_user_detail.member_status` | `member_status` | |
| `s15_v3_user_detail.opt_in_status` | `opt_in_status` | |
| `s15_v3_user_detail.opt_in_out_date` | `opt_in_out_date` | |
| `s15_v3_user_detail.opt_reason` | `opt_reason` | |
| `s15_v3_user_detail.source` | `source` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL columns (`user`):** `user_id` (internal PK replaced by UUID), `multitemplate_id` — see global note.

**Unmapped MySQL columns (`s15_v3_user_detail`):** `id`, `user_id` (join key), `multitemplate_id` — see global note, `available_points`, `lifetime_points`, `used_points`, `expire_points` (computed/denormalized), `next_inactive_date` (derived), `create_date`, `update_date`.

**Unmapped PostgreSQL columns:** `id` (auto-generated UUID), `member_since`, `deleted_at`, `custom_attributes` (populated from EAV — see Appendix).
</details>

---

### 2.2 `s15_v3_user_blacklist` → `blacklisted_users`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `uid` | `member_id` | |
| `blocked_by` | `blocked_by` | |
| `create_date` | `created_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `id`, `multitemplate_id` — see global note.
**Unmapped PostgreSQL:** `id` (auto-generated UUID), `is_active` (defaults to `true`), `updated_at`.
</details>

---

### 2.3 `s15_v3_domain_blacklist` → `blacklisted_domains`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `domain` | `domain` | |
| `blocked_by` | `blocked_by` | |
| `create_date` | `created_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `id`, `multitemplate_id` — see global note.
**Unmapped PostgreSQL:** `id` (auto-generated UUID), `reason` (defaults to `NULL`), `is_active` (defaults to `true`), `updated_at`.
</details>

---

### 2.4 `s15_v3_merge_user_detail` → `merge_user_requests`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `primary_uid` | `primary_member_id` | |
| `secondary_uid` | `secondary_member_id` | |
| `merged_by` | `merged_by` | |
| `reason` | `reason` | |
| `duplicate_actions` | `duplicate_actions` | `TEXT → JSONB` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `id`, `user_id`, `multitemplate_id` — see global note, `primary_user_mt_id`, `secondary_user_mt_id`, `transaction_ids`, `transaction_uuids`, `process_status`, `process_note`, `raf_status`.
**Unmapped PostgreSQL:** `id` (auto-generated UUID).
</details>

---

### 2.5 `s15_v3_user_rewards` → `reward_members`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | *(via `reward_id` FK)* | Site resolved through reward |
| `transaction_id` | `transaction_id` | Lookup UUID via `old_transaction_id` |
| `user_id` | `user_id` | Lookup UUID via `member_id` |
| `reward_id` | `reward_id` | Lookup UUID via `migrated_reward_id` |
| `issuance_details` | `issuance_details` | `TEXT → JSONB` |
| `created_at` | `created_at` | |
| `updated_at` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `id`, `multitemplate_id` — see global note, `transaction_uuid`.
**Unmapped PostgreSQL:** `id` (auto-generated UUID), `transaction_created_at` (derived from transaction).
</details>

---

### 2.6 `s15_v3_user_inactive_status_history` → `member_statuses`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `user_id` | `user_id` | Lookup UUID via `member_id` |
| `member_current_status` | `current_status` | |
| `next_inactive_date` | `next_inactive_date` | |
| `next_member_status` | `next_status` | |
| `current_milestone` | `current_milestone` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `id`, `multitemplate_id` — see global note.
**Unmapped PostgreSQL:** `id` (auto-generated UUID).
</details>

---

## 3. Actions

### Table Mapping Summary

| MySQL Table | PostgreSQL Table |
|---|---|
| `s15_v3_action_detail` | `actions` |
| `s15_v3_action_group` | `action_groups` |
| `s15_v3_action_group_association` | `action_group_actions` |
| `s15_v3_action_series` | `action_series` |
| `s15_v3_action_series_map` | `action_series_actions` |
| `s15_v3_purchase_transaction_types` | `action_item_type_settings` |
| `s15_v3_standard_action_list` | `action_masters` |
| `s15_v3_actionseries_earned` | `action_series_members` |

---

### 3.1 `s15_v3_action_detail` → `actions`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `action_id` | `action_id` | |
| `action_name` | `name` | |
| `action_name_display` | `display_name` | |
| `action_points` | `points` | |
| `ratio` | `multiplier` | |
| `status` | `active` | `enum: {0:false, 1:true}` |
| `hold_points_flag` | `hold_type` | `enum: {0:0, 1:1, 2:2, 3:0}` |
| `hold_days` | `hold_days` | |
| `max_points` | `points_limit` | |
| `action_limit_type` | `limit_type` | `enum: {0:ROLLING, 1:CALENDAR}` |
| `calendar_period` | `period_unit` | `enum: {0:null, 1:DAY, 2:WEEK, 3:MONTH, 4:YEAR, 5:ANNIVERSARY}` |
| `period` | `period_value` | |
| `interval_period_type` | `interval_unit` | `enum: {0:null, 1:DAYS, 2:HOURS}` |
| `interval_period_limit` | `interval_value` | |
| `validate_action_attribute` | `validate_attributes` | `enum: {0:false, 1:true}` |
| `points_expiration_type` | `expiration_type` | `enum: {0:ROLLING, 1:CALENDAR}` |
| `calendar_expire` | `expiration_unit` | `enum: {0:null, 1:WEEK, 2:MONTH, 3:YEAR}` |
| `expire_in_days` | `expiration_value` | |
| `transaction_type_flag` | `is_item_type_multiplier_enabled` | `enum: {0:false, 1:true}` |
| `tier_based_limit` | `tier_limit_enabled` | `enum: {0:false, 1:true}` |
| `auto_delivery_overwrite` | `auto_delivery_override` | `enum: {0:false, 1:true}` |
| `minimum_value` | `auto_delivery_minimum_threshold` | |
| `tier_based_expiration` | `tier_expiration_enabled` | `enum: {0:false, 1:true}` |
| `max_order_apply_hold` | `max_order_limit_hold_enabled` | `enum: {0:false, 1:true}` |
| `max_order_award_type` | `order_points_award_type` | `enum: {0:null, 1:ZEROPOINTS, 2:POINTS}` |
| `max_order_type` | `order_limit_scope` | `enum: {0:null, 1:PERCENTAGE, 2:POINTS, 3:SPENDAMOUNT}` |
| `max_order_value` | `order_limit_value` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `id`, `multitemplate_id` — see global note, `enable_points_differentiation_by_action_attribute`, `action_attribute_id`, `action_desc`, `action_active_url`, `action_inactive_url`, `action_points_display`, `action_earned_name_display`, `action_limit_display`, `purchase_point_type`, `apply_all_ratios_in_flat_points`, `enable_disable`, `points_expire_year_value`, `disable_large_value_order_points`, `large_value_order_from`, `large_value_order_to`, `minimum_spend_amount`, `custom_range_flag`, `button_title`, `action_redirect_url`, `display_action_sequence`, `action_api_display`, `max_order_limit_flag`, `include_customid_parameter`, `repeat_customid_parameter`, `include_actionlimit_in_customid`.
**Unmapped PostgreSQL:** `id` (auto-generated UUID), `is_deleted`, `deleted_at`.
</details>

---

### 3.2 `s15_v3_action_group` → `action_groups`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `group_id` | `migrated_action_group_id` | |
| `group_name` | `name` | |
| `group_active_badge_url` | `active_badge_url` | |
| `group_inactive_badge_url` | `inactive_badge_url` | |
| `group_max_limit` | `points_limit` | |
| `group_action_limit_type` | `limit_type` | `enum: {0:ROLLING, 1:CALENDAR}` |
| `group_calendar_period` | `period_unit` | `enum: {0:null, 1:DAY, 2:WEEK, 3:MONTH, 4:YEAR}` |
| `group_rolling_period` | `period_value` | |
| `group_action_status` | `is_active` | `enum: {0:false, 1:true}` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `multitemplate_id` — see global note, `grouped_action_id` (relationships stored in `action_group_actions`), `id`.
**Unmapped PostgreSQL:** `id` (auto-generated UUID, or from MySQL `id` field if present).
</details>

---

### 3.3 `s15_v3_action_group_association` → `action_group_actions`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `group_id` | `action_group_id` | Lookup `action_groups.id` via `migrated_action_group_id` |
| `action_id` | `action_id` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `id`.
**Unmapped PostgreSQL:** `id` (auto-generated UUID).
</details>

---

### 3.4 `s15_v3_action_series` → `action_series`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `action_series_id` | `migrated_action_series_id` | |
| `action_series_name` | `name` | |
| `series_expire_type` | `duration_type` | |
| `series_expire_days` | `duration_days` | |
| `series_expire_recurring_type` | `duration_mode` | `enum: {0:null, 1:DAYS, 2:WEEKS, 3:MONTHS, 4:YEARS, 5:ANNIVERSARY, 6:DATE_RANGE}` |
| `series_start_date` | `duration_start_date` | |
| `series_end_date` | `duration_end_date` | |
| `action_series_status` | `is_active` | `enum: {0:false, 1:true}` |
| `badge_url` | `active_badge_url` | |
| `inactive_badge_url` | `inactive_badge_url` | |
| `description` | `description` | |
| `tagline` | `tagline` | |
| `terms_and_condition` | `terms` | |
| `enable_series_milestone_limit` | `milestone_limit_enabled` | `enum: {0:false, 1:true}` |
| `action_series_type` | `evaluation_type` | `enum: {0:COUNTS, 1:POINTS, 2:COUNT_AND_POINTS}` |
| `action_series_type_min_points` | `minimum_threshold` | |
| `points` | `bonus_points` | |
| `hold_points` | `hold_points_days` | |
| `points_expire_type` | `expiration_type` | |
| `points_expire_days` | `expiration_unit` | |
| `points_expire_year_value` | `expiration_value` | |
| `maximum_achievement_limit_value` | `max_achievement_limit` | |
| `maximum_achievement_limit_recurring_type` | `max_achievement_limit_period` | `enum: {0:null, 1:DAY, 2:WEEK, 3:MONTH, 4:YEAR, 5:ANNIVERSARY}` |
| `maximum_point_limit_value` | `max_point_limit` | |
| `maximum_point_limit_recurring_type` | `max_point_limit_period` | `enum: {0:null, 1:DAY, 2:WEEK, 3:MONTH, 4:YEAR, 5:ANNIVERSARY}` |
| `reward_id` | `benefit_reward_id` | Lookup UUID via `migrated_reward_id` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `multitemplate_id` — see global note, `points_expire`, `benefit_type`, `campaign_id`, `minimum_amount`, `coupon_id`.
**Unmapped PostgreSQL:** `id` (auto-generated UUID).
</details>

---

### 3.5 `s15_v3_action_series_map` → `action_series_actions`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | *(derived from action_series)* | |
| `action_series_id` | `action_series_id` | Lookup `action_series.id` via `migrated_action_series_id` |
| `action_id` | `action_id` | |
| `threshold_points` | `action_count` | |
| `minimum_amount_type` | `calculation_type` | `enum: {Only Count:ONLY_COUNT, Per Order:PER_ORDER, Total Purchase:TOTAL_PURCHASE, Base On Spend:BASE_ON_SPEND, Base On Point:BASE_ON_POINT}` |
| `minimum_amount` | `criteria_value` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `id`, `multitemplate_id` — see global note, `action_use`.
**Unmapped PostgreSQL:** `id` (auto-generated UUID).
</details>

---

### 3.6 `s15_v3_purchase_transaction_types` → `action_item_type_settings`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `transaction_type_id` | `migrated_transaction_type_id` | |
| `transaction_type` | `item_type` | |
| `ratio` | `multiplier` | |
| `points_expiration_type` | `expiration_type` | `enum: {0:ROLLING, 1:CALENDAR}` |
| `calendar_expire` | `expiration_unit` | `enum: {0:null, 1:WEEK, 2:MONTH, 3:YEAR}` |
| `expire_in_days` | `expiration_value` | Used when `expiration_type=ROLLING` |
| `points_expire_year_value` | `expiration_value` | Used when `expiration_type=CALENDAR` and `unit=YEAR` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `multitemplate_id` — see global note, `incentive_id`.
**Unmapped PostgreSQL:** `id` (auto-generated UUID), `purchase_action_id`.
</details>

---

### 3.7 `s15_v3_standard_action_list` → `action_masters`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `action_id` | `action_id` | |
| `action_name` | `name` | |
| `is_internal_standard_action` | `is_internal` | `enum: {0:false, 1:true}` |
| `create_date` | `created_at` | |

---

### 3.8 `s15_v3_actionseries_earned` → `action_series_members`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `action_series_id` | `action_series_id` | Lookup `action_series.id` via `migrated_action_series_id` |
| `user_id` | `user_id` | Lookup `users.id` via `uid → member_id` |
| `transaction_id` | `transaction_id` | Lookup `transactions.id` via `migrated_transaction_id` |
| `status` | `benefit_status` | `enum: {0:GRANTED, 1:REVOKED}` |
| `benefit_type` | `benefit_type` | `enum: {0:BONUS, 1:REWARD}` |
| `coupon` | `reward_code` | |
| `duration_cycle_start_date` | `cycle_start_at` | |
| `duration_cycle_end_date` | `cycle_end_at` | |
| `created_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped PostgreSQL:** `id` (auto-generated UUID), `transaction_created_at` (no MySQL source column — fetched from `transactions.created_at`).
</details>

---

## 4. Campaigns

### Table Mapping Summary

| MySQL Table | PostgreSQL Table | Notes |
|---|---|---|
| `s15_v3_campaign` | `campaigns` | |
| `s15_v3_campaign_detail` | `campaign_milestones` | |
| `s15_v3_campaign_detail` | `campaign_milestone_tiers` | Split from same source; `tier_id` CSV expanded |
| `s15_v3_campaign_group` | `campaign_groups` | |
| `s15_v3_campaign_group_association` | `campaign_groups_campaigns` | |
| `s15_v3_user_validity_and_activation_campaign` | `campaign_user_activations` | |
| `s15_v3_campaign_user_track` | `campaign_benefits` | |
| `s15_v3_transaction_type_campaign_detail` | `campaign_action_item_type_settings` | |

---

### 4.1 `s15_v3_campaign` → `campaigns`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `id` | `migrated_campaign_id` | |
| `campaign_name` | `name` | |
| `activation_required` | `activation_required` | `enum: {0:false, 1:true}` |
| `individual_validity` | `per_user_validity` | `enum: {0:false, 1:true}` |
| `start_datetime` | `start_date` | |
| `end_datetime` | `end_date` | |
| `active` | `is_active` | `enum: {0:false, 1:true}` |
| `campaign_benefit_type_id` | `benefit_type` | `enum: {0:null, 1:reward, 3:point}` |
| `reward_id` | `benefit_reward_id` | Lookup UUID via `migrated_reward_id` |
| `campaign_benefit_value` | `bonus_points` | |
| `campaign_image_url` | `image_url` | |
| `campaign_desc` | `description` | |
| `campaign_tagline` | `tagline` | |
| `campaign_terms` | `terms` | |
| `campaign_benefit_image_url` | `benefit_image_url` | |
| `maximum_achievement_limit_value` | `max_achievement_limit` | |
| `maximum_achievement_limit_flag` | `max_achievement_limit_period` | `enum: {0:null, 1:DAY, 2:WEEK, 3:MONTH, 4:YEAR, 5:ANNIVERSARY}` |
| `maximum_point_limit_value` | `max_point_limit` | |
| `maximum_point_limit_flag` | `max_point_limit_period` | `enum: {0:null, 1:DAY, 2:WEEK, 3:MONTH, 4:YEAR, 5:ANNIVERSARY}` |
| `apply_one_milestone_individual_validity` | `single_milestone_per_validity_period` | `enum: {0:false, 1:true}` |
| `campaign_benefit_name` | `benefit_name` | |
| `campaign_points_expire_type` | `expiration_type` | `enum: {0:ROLLING, 1:CALENDAR}` |
| `points_expire_year_value` | `expiration_unit` | `enum: {0:null, 1:WEEK, 2:MONTH, 3:YEAR}` |
| `campaign_points_expire_value` | `expiration_value` | |
| `shipment_status` | `hold_type` | `enum: {0:null, 1:SHIPMENT, 2:CAMPAIGN_COMPLETION}` |
| `campaign_benefit_hold_days` | `hold_days` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `multitemplate_id` — see global note, `campaign_group_name`, `segment_id`, `tier_id`, `campaign_benefit_expiration_from/to/type/period_days/count`, `maximum_point_limit_duration/days`, `coupon_maximum_achievement_limit_value/option`.
**Unmapped PostgreSQL:** `id` (auto-generated UUID), `custom_attributes`.
</details>

---

### 4.2 `s15_v3_campaign_detail` → `campaign_milestones`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `uuid` | `id` | MySQL `uuid` field → PG primary key |
| `site_id` | `site_id` | |
| `id` | `migrated_milestone_id` | |
| `milestone_name` | `name` | |
| `campaign_id` | `campaign_id` | Lookup `campaigns.id` via `migrated_campaign_id` |
| `segment_id` | `segment_id` | Lookup `segments.id` via `migrated_segment_id` |
| `milestone_type_id` | `milestone_type` | `enum: {0:null, 1:ACTION, 2:ACTION_SERIES}` |
| `milestone_detail_type_id` | `action_id` / `action_series_id` | Based on `milestone_type_id` (1→`action_id`, 2→`action_series_id`) |
| `milestone_bonus_type_flag` | `bonus_type` | `enum: {0:null, 1:point, 2:ratio, 3:point_additional}` |
| `milestone_benefit_value` | `bonus_points` / `multiplier` | Based on `milestone_bonus_type_flag` |
| `milestone_desc` | `description` | |
| `milestone_active_image_url` | `active_image_url` | |
| `milestone_inactive_image_url` | `inactive_image_url` | |
| `milestone_maximum_achievement_limit_value` | `max_achievement_limit` | |
| `milestone_maximum_achievement_limit_flag` | `max_achievement_limit_period` | `enum: {0:null, 1:DAY, 2:WEEK, 3:MONTH, 4:YEAR, 5:CAMPAIGN}` |
| `milestone_maximum_point_limit_value` | `max_point_limit` | |
| `milestone_maximum_point_limit_flag` | `max_point_limit_period` | `enum: {0:null, 1:DAY, 2:WEEK, 3:MONTH, 4:YEAR, 5:CAMPAIGN}` |
| `milestone_product_id_based_bonus_flag` | `is_product_specific` | `enum: {0:false, 1:true}` |
| `milestone_points_expire_type` | `expiration_type` | `enum: {0:ROLLING, 1:CALENDAR}` |
| `points_expire_year_value` | `expiration_unit` | `enum: {0:null, 1:WEEK, 2:MONTH, 3:YEAR}` |
| `milestone_points_expire_value` | `expiration_value` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `multitemplate_id` — see global note, `milestone_benefit_type_id`, `milestone_benefit_type_flag`, `milestone_benefit_expiration_from/to/period/days/count`, `milestone_bonus`, `running_order_bonus`, `org_action_id`, `override_action_id`, `tier_id` (expanded into `campaign_milestone_tiers`).
</details>

---

### 4.3 `s15_v3_campaign_detail` → `campaign_milestone_tiers`

> Split from the same MySQL source as `campaign_milestones`. The `tier_id` column is a comma-separated list that is expanded into individual rows.

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `id` (milestone) | `campaign_milestone_id` | Lookup `campaign_milestones.id` via `migrated_milestone_id` |
| `tier_id` (CSV) | `tier_id` | Split CSV; lookup `tiers.id` via `migrated_tier_id` for each value |

<details>
<summary>Unmapped columns</summary>

**Unmapped PostgreSQL:** `id` (auto-generated UUID), `created_at`, `updated_at`.
</details>

---

### 4.4 `s15_v3_campaign_group` → `campaign_groups`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `id` (varchar UUID) | `id` | |
| `campaign_group_id` | `migrated_campaign_group_id` | |
| `campaign_group_name` | `name` | |
| `campaign_group_precedence_rule` | `precedence_rule` | |
| `campaign_group_status` | `is_active` | `enum: {0:false, 1:true}` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 4.5 `s15_v3_campaign_group_association` → `campaign_groups_campaigns`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `campaign_group_id` | `campaign_group_id` | Lookup `campaign_groups.id` via `migrated_campaign_group_id` |
| `campaign_id` | `campaign_id` | Lookup `campaigns.id` via `migrated_campaign_id` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |
| *(no MySQL source)* | `sequence` | No MySQL source; default `0` |

---

### 4.6 `s15_v3_user_validity_and_activation_campaign` → `campaign_user_activations`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `id` | `migrated_user_activation_id` | |
| `campaign_id` | `campaign_id` | Lookup `campaigns.id` via `migrated_campaign_id` |
| `user_id` | `user_id` | Lookup `users.id` via `uid → member_id` |
| `individual_validity_start_date` | `start_date` | |
| `individual_validity_end_date` | `end_date` | |
| `activated_on` | `activated_at` | |
| `activated` | `is_active` | `enum: {0:false, 1:true}` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 4.7 `s15_v3_campaign_user_track` → `campaign_benefits`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `user_id` | `user_id` | Lookup `users.id` via `member_id` |
| `campaign_id` | `campaign_id` | Lookup `campaigns.id` via `migrated_campaign_id` |
| `milestone_id` | `campaign_milestone_id` | Lookup `campaign_milestones.id` via `migrated_milestone_id` |
| `transaction_id` | `transaction_id` | Lookup `transactions.id` via `old_transaction_id` |
| `refrence_transaction_id` | `reference_transaction_id` | Lookup `transactions.id` via `old_transaction_id` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `id`, `multitemplate_id` — see global note, `reference_action_id`, `individual_validity_id`, `transaction_uuid`, `refrence_transaction_uuid`, `benefit`.
**Unmapped PostgreSQL:** `id`, `transaction_created_at`, `campaign_group_id`, `multiplier`, `base_points`, `points_awarded`, `bonus_points`, `total_points_earned`, `reward_id`, `migrated_campaign_benefit_id`, `campaign_user_activation_id`.
</details>

---

### 4.8 `s15_v3_transaction_type_campaign_detail` → `campaign_action_item_type_settings`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `campaign_id` | `campaign_id` | Lookup `campaigns.id` via `migrated_campaign_id` |
| `milestone_campaign_id` | `campaign_milestone_id` | Lookup `campaign_milestones.id` via `migrated_milestone_id` |
| `transaction_type_id` | `action_item_type_setting_id` | Lookup `action_item_type_settings.id` via `migrated_transaction_type_id` |
| `transaction_type_campaign_ratio` | `multiplier` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

## 5. Badges

### Table Mapping Summary

| MySQL Table(s) | PostgreSQL Table | Notes |
|---|---|---|
| `s15_v3_badges` | `badges` | |
| `s15_v3_badge_images` | `badges` | `images` JSONB column |
| `s15_v3_badge_attributes` + `s15_v3_badge_attribute_values` | `badges` | `custom_attributes` JSONB column |
| `s15_v3_badge_groups` | `badge_groups` | |
| `s15_v3_user_badges` | `badge_members` | |
| `s15_v3_badge_summary` | `badge_stats` | |

---

### 5.1 `s15_v3_badges` → `badges`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `id` (varchar 50) | `id` | |
| `site_id` | `site_id` | |
| `badge_group_id` | `badge_group_id` | Lookup `badge_groups.id` |
| `badge_code` | `code` | |
| `badge_name` | `name` | |
| `display_name` | `display_name` | |
| `limited_availability` | `limited_availability` | `enum: {0:false, 1:true}` |
| `start_date` | `start_date` | |
| `end_date` | `end_date` | |
| `badge_status` | `status` | `enum: {1:draft, 2:scheduled, 3:live, 4:archived}` |
| `created_at` | `created_at` | |
| `updated_at` | `updated_at` | |
| *(from `s15_v3_badge_images`)* | `images` | JSONB array — see Appendix |
| *(from `s15_v3_badge_attributes` + `s15_v3_badge_attribute_values`)* | `custom_attributes` | JSONB — see Appendix |
| `apply_limit` | `apply_limit` | `enum: {0:false, 1:true}`; default `false` |
| `limit_type` | `limit_type` | string; default `TOTAL_COUNT` |
| `limit_count` | `limit_count` | integer; default `0` |
| `allow_requalify` | `allow_requalify` | `enum: {0:false, 1:true}`; default `false` |
| `requalify_type` | `requalify_type` | nullable string |
| `requalify_limit` | `requalify_limit` | nullable integer |
| `expire_badge` | `expire_badge` | nullable string |
| `expiry_type` | `expiry_type` | nullable string |
| `expiry_value` | `expiry_value` | nullable integer |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `multitemplate_id` — see global note.
</details>

---

### 5.2 `s15_v3_badge_groups` → `badge_groups`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `id` (varchar 50) | `id` | |
| `site_id` | `site_id` | |
| `badge_group_code` | `code` | |
| `badge_group_name` | `name` | |
| `badge_group_description` | `description` | |
| `created_at` | `created_at` | |
| `updated_at` | `updated_at` | |

---

### 5.3 `s15_v3_user_badges` → `badge_members`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `id` (varchar 50) | `id` | |
| `site_id` | `site_id` | |
| `badge_id` | `badge_id` | Lookup `badges.id` |
| `user_id` | `user_id` | Lookup `users.id` via `member_id` |
| `user_badge_source` | `source` | |
| `source_id` | `source_id` | |
| `source_name` | `source_name` | |
| `note` | `note` | |
| `expire_at` | `expiry_date` | |
| `created_at` | `created_at` | |
| `updated_at` | `updated_at` | |

---

### 5.4 `s15_v3_badge_summary` → `badge_stats`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `id` (varchar 50) | `id` | |
| `site_id` | `site_id` | |
| `badge_id` | `badge_id` | |
| `issued_count` | `issued_count` | |
| `unique_members_count` | `unique_members_count` | |
| `created_at` | `created_at` | |
| `updated_at` | `updated_at` | |

---

## 6. Tiers

### Table Mapping Summary

| MySQL Table | PostgreSQL Table | Notes |
|---|---|---|
| `s15_v3_tier` | `tiers` | |
| `s15_v3_tier_extended_attribute` | `tiers` | `custom_attributes` JSONB column |
| `s15_v3_tier_earned` | `tier_earned_members` | |
| `s15_v3_user_tier_tracking` | `tier_earned_histories` | |
| `s15_v3_tier_based_rewards_limit` | `reward_tier_rules` | |
| `s15_v3_tier_benefit_details` | `tier_benefits` | |
| `s15_v3_tier_custom_bracket` | `tier_purchase_brackets` | |
| `s15_v3_tier_metrics` | `tier_metrics` | |
| `s15_v3_hierarchy_group_tier_earned` | `tier_earned_groups` | |
| `s15_v3_tier_bonus_tracking` | `tier_benefit_transactions` | |

---

### 6.1 `s15_v3_tier` → `tiers`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `tier_id` | `migrated_tier_id` | |
| `id` (varchar 50) | `id` | |
| `program_id` | `program_id` | Lookup `programs.id` |
| `tier_name` | `name` | |
| `purchase_ratio` | `multiplier` | |
| `bonus_points` | `bonus_points` | |
| `tier_type` | `rule_type` | |
| `threshold_earned_points` | `threshold` | |
| `tier_display_name` | `display_name` | |
| `active_status` | `is_active` | `enum: {0:false, 1:true}` |
| `grouptier` | `group_applicable` | `enum: {0:false, 1:true}` |
| `bracket_points_flag` | `purchase_benefit_type` | `enum: {0:DEFAULT, 1:STANDARD, 2:CUSTOM}` |
| `action_points_expiration_type` | `action_expiration_type` | `enum: {0:ROLLING, 1:CALENDAR}` |
| `rolling_expire` | `action_expiration_value` | Used when `action_expiration_type=ROLLING` |
| `calendar_expire` | `action_expiration_unit` | `enum: {0:null, 1:WEEK, 2:MONTH, 3:YEAR}` |
| `points_expire_year_value` | `action_expiration_value` | Used when `action_expiration_type=CALENDAR` |
| `bonus_points_expire_type` | `bonus_expiration_type` | |
| `bonus_rolling_expire` | `bonus_expiration_value` | |
| `bonus_calendar_expire` | `bonus_expiration_unit` | |
| `bonus_points_expire_year_value` | `bonus_expiration_value` | |
| `tier_based_limit` | `action_limit_override` | |
| `max_points` | `action_points_limit` | |
| `period` | `action_period_value` | |
| `calendar_period` | `action_period_unit` | |
| `tier_limit_type` | `action_limit_type` | |
| `tier_downgrade_maintain_period` | `downgrade_in_maintain_period` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL:** `multitemplate_id` — see global note, `tier_display_id`, `tier_entry_condition`, `tier_exit_condition`, `tier_time_dimension_id`, `tier_retention_period`, `tier_retention_type`, `tier_color_code`, `tier_additional_info`, `tier_setup_status`, `tier_version_status`, `threshold_min/max_multiplier`, `action_series_id`, `bonus_points_achievement`, `priority`.
**Unmapped PostgreSQL:** `expire_date`, `level`, `entry_condition` (JSONB), `retention_length`, `retention_interval`, `tier_v2`, `data` (JSONB), `bonus_repeatable`, `custom_attributes` (EAV — see Appendix).
</details>

---

### 6.2 `s15_v3_tier_earned` → `tier_earned_members`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `user_id` | `user_id` | Lookup `users.id` via `member_id` |
| `tier_id` | `tier_id` | Lookup `tiers.id` via `migrated_tier_id` |
| `program_id` | `program_id` | Lookup `programs.id` |
| `reason` | `reason` | |
| `expire_date` | `expiry_date` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 6.3 `s15_v3_user_tier_tracking` → `tier_earned_histories`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `user_id` | `entity_id` | Lookup `users.id`; `entity_type='user'` |
| `tier_id` | `tier_id` | Lookup `tiers.id` via `migrated_tier_id` |
| `program_id` | `program_id` | Lookup `programs.id` |
| `manual_tier_assign` + `type` | `action` | `enum: {upgrade, downgrade, maintain}` |
| `create_date` | `earned_at` | |

---

### 6.4 `s15_v3_tier_based_rewards_limit` → `reward_tier_rules`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `reward_id` | `reward_id` | Lookup `rewards.id` via `migrated_reward_id` |
| `tier_id` | `tier_id` | Lookup `tiers.id` via `migrated_tier_id` |
| `tier_redemption_eligible` | `is_eligible` | `enum: {0:false, 1:true}` |
| `redemption_limit_period` | `limit_period` | |
| `redemption_limit` | `limit_value` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 6.5 `s15_v3_tier_benefit_details` → `tier_benefits`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `id` (varchar 40) | `id` | |
| `site_id` | `site_id` | |
| `program_id` | `program_id` | |
| `tier_id` | `tier_id` | |
| `tier_benefit_details_id` | `tier_benefit_details_id` | |
| `tier_benefit_type` | `benefit_type` | |
| `tier_benefit_sub_type` | `benefit_sub_type` | |
| `tier_benefit_value` | `benefit_value` | |
| `tier_benefit_expire_type` | `expiration_type` | |
| `tier_benefit_rolling_expire_days` | `rolling_expire_days` | |
| `tier_benefit_calendar_expire_type` | `calendar_expire_type` | |
| `tier_benefit_calendar_expire_value` | `calendar_expire_value` | |
| `tier_benefit_action_limit_type` | `action_limit_type` | |
| `tier_benefit_action_limit_rolling_days` | `action_limit_rolling_days` | |
| `tier_benefit_action_limit_calendar_type` | `action_limit_calendar_type` | |
| `tier_benefit_action_limit_calendar_value` | `action_limit_calendar_value` | |
| `tier_benefit_interval_period_type` | `interval_period_type` | |
| `tier_benefit_interval_period_limit` | `interval_period_limit` | |
| `tier_benefit_start_date` | `start_date` | |
| `tier_benefit_status` | `is_active` | |
| `created_at` | `created_at` | |
| `updated_at` | `updated_at` | |

---

### 6.6 `s15_v3_tier_custom_bracket` → `tier_purchase_brackets`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `tier_id` | `tier_id` | Lookup `tiers.id` via `migrated_tier_id` |
| `lower_limit` | `min_amount` | |
| `upper_limit` | `max_amount` | |
| `points_awarding_type` | `points_type` | `enum: {1:RATIO, 2:FLAT_POINTS}` |
| `purchase_ratio` | `points_value` | Used when `points_type=RATIO` |
| `flat_points` | `points_value` | Used when `points_type=FLAT_POINTS` |
| `db_add_date` | `created_at` | |
| `db_update_date` | `updated_at` | |

---

### 6.7 `s15_v3_tier_metrics` → `tier_metrics`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `id` (varchar 40) | `id` | |
| `site_id` | `site_id` | |
| `metrics_name` | `name` | |
| `metrics_formula` | `formula` | `JSON → JSONB` |
| `metrics_status` | `active` | `enum: {0:false, 1:true}` |
| `created_at` | `created_at` | |
| `updated_at` | `updated_at` | |

---

### 6.8 `s15_v3_hierarchy_group_tier_earned` → `tier_earned_groups`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `group_id` | `group_id` | Lookup `groups.id` via `migrated_group_id` |
| `tier_id` | `tier_id` | Lookup `tiers.id` via `migrated_tier_id` |
| `program_id` | `program_id` | Lookup `programs.id` |
| `reason` | `reason` | |
| `expire_date` | `expiry_date` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 6.9 `s15_v3_tier_bonus_tracking` → `tier_benefit_transactions`

**Sources:** `s15_v3_tier_bonus_tracking` (primary), `user` (joined on `user_id` + `site_id` to resolve `uid`)

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `user.uid` | `user_id` | Lookup `users.id` via `uid → member_id` (joined from `user` table) |
| `tier_id` | `tier_id` | Lookup `tiers.id` via `migrated_tier_id` |
| *(derived)* | `program_id` | Fetched from `tiers.program_id` WHERE `tiers.id = tier_id`; no direct MySQL column |
| `tier_benefit_detail_id` | `tier_benefit_id` | FK to `tier_benefits`; no `migrated_tier_benefit_id` stored — comparison skipped |
| `transaction_id` | `transaction_id` | Lookup `transactions.id` via `migrated_transaction_id` |
| *(derived)* | `transaction_created_at` | Fetched from `transactions.created_at` WHERE `transactions.id = transaction_id`; no direct MySQL column |
| `db_add_date` | `created_at` | |
| `db_update_date` | `updated_at` | |

---

## 7. Groups (Hierarchy)

### Table Mapping Summary

| MySQL Table | PostgreSQL Table |
|---|---|
| `s15_v3_hierarchy_groups` | `groups` |
| `s15_v3_hierarchy_group_details` | `group_members` |
| `s15_v3_hierarchy_group_invitations` | `group_invitations` |
| `s15_v3_hierarchy_group_configurations` | `group_settings` |
| `s15_v3_hierarchy_group_leave_history` | `group_membership_histories` |

---

### 7.1 `s15_v3_hierarchy_groups` → `groups`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `group_id` | `migrated_group_id` | |
| `id` (varchar 50) | `id` | |
| `group_name` | `name` | |
| `created_by` | `created_by_id` | Lookup `users.id` via `member_id` |
| `updated_by` | `updated_by_id` | Lookup `users.id` via `member_id` |
| `admin_user` | `admin_user_id` | Lookup `users.id` via `member_id` |
| `group_status` | `status` | `enum: {0:false, 1:true}` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 7.2 `s15_v3_hierarchy_group_details` → `group_members`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `group_id` | `group_id` | Lookup `groups.id` via `migrated_group_id` |
| `user_id` | `user_id` | Lookup `users.id` via `member_id` |
| `invited_by` | `invited_by` | Lookup `users.id` via `member_id` |
| `user_role` | `role` | `enum: {1:owner, 2:member, default:member}` |
| `member_status` | `status` | `enum: {1:active, 0:inactive}` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 7.3 `s15_v3_hierarchy_group_invitations` → `group_invitations`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `group_id` | `group_id` | Lookup `groups.id` via `migrated_group_id` |
| `user_id` | `user_id` | Lookup `users.id` via `member_id` (invitee) |
| `admin_user_id` | `invited_by` | Lookup `users.id` via `member_id` |
| `invitations_accept_status` | `status` | `enum: {0:invited, 1:accepted, 2:declined}` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 7.4 `s15_v3_hierarchy_group_configurations` → `group_settings`

**Sources:** `s15_v3_hierarchy_group_configurations` (per-group settings), `s15_v3_hierarchy_settings` (site-default)

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `group_id` | *(via `group_setting_assignments`)* | Group-level setting assignment |
| `allow_auto_transfer_points_to_group` | `auto_transfer_points_enabled` | `enum: {0:false, 1:true}` |
| `allow_auto_transfer_points_on_joining` | `auto_transfer_on_join_enabled` | `enum: {0:false, 1:true}` |
| `enable_auto_group_points_redemption` | `auto_redemption_enabled` | `enum: {0:false, 1:true}` |
| `allow_group_points_redemption` | `redemption_allowed_for` | `enum: {0:NONE, 1:OWNER}` |
| `group_invitation_acceptance_required` | `invitation_acceptance_required` | `enum: {0:false, 1:true}` |
| `restrict_members_to_leave_group` | `restrict_member_exit` | `enum: {0:false, 1:true}` |
| `group_deactivation_rule` | `dissolution_allowed_for` | `enum: {1:OWNER, 2:ADMIN}` |
| `points_distribution_on_group_deactivation` | `dissolution_points_distribution` | `enum: {1:TRANSFER_TO_DEACTIVATOR, 2:REDISTRIBUTE}` |
| `max_member_to_the_group` | `enforce_member_limit` | `enum: {0:false, 1:true}` |
| `max_member_limit` | `max_member_limit` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 7.5 `s15_v3_hierarchy_group_leave_history` → `group_membership_histories`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `group_id` | `group_id` | Lookup `groups.id` via `migrated_group_id` |
| `user_id` | `user_id` | Lookup `users.id` via `member_id` |
| `approved_by` | `performed_by` | Lookup `users.id` via `member_id` |
| `request_status` | `action` | `enum: {0:leave_requested, 1:left, 2:leave_rejected}` |
| `create_date` | `created_at` / `performed_at` | |
| `update_date` | `updated_at` | |

---

## 8. Segments

### Table Mapping Summary

| MySQL Table | PostgreSQL Table | Notes |
|---|---|---|
| `s15_v3_segment` | `segments` | Many MySQL rule columns go to `segment_rules` (PG-only) |
| `s15_v3_segment_user_log` | `segment_members` | Filter: `is_eligible = 1` only |
| `s15_v3_issuance_segment_purchase_qualified` | `segment_qualified_transactions` | |
| `s15_v3_segment_and_or_mapping` | `segment_rule_groups` | Filter: `section_and_or_group = 0` only |

---

### 8.1 `s15_v3_segment` → `segments`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `id` (varchar 50) | `id` | |
| `segment_id` | `migrated_segment_id` | |
| `segment_name` | `name` | |
| `description` | `description` | |
| `tagline` | `tagline` | |
| `terms_and_condition` | `terms_and_conditions` | |
| `status` | `is_active` | `enum: {0:false, 1:true}` |
| `db_add_date` | `created_at` | |
| `db_update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL (rule columns — stored in `segment_rules` PG table):** `segment_type`, `segment_from_date`, `segment_to_date`, `anniversary_date_check`, `birth_date_check`, `available_points_from/to/flag`, `lifetime_points_from/to/flag`, `purchase_amount_from/to`, `age_from/to`, `gender`, `city`, `store_level_1` through `store_level_10`, and ~30 additional rule-condition columns.

**Unmapped MySQL (operational):** `multitemplate_id` — see global note, `is_activate_leaderboard`, `users_segment_id`, `segment_migration_status`, `prev_segment_id`, `clone_status`, `member_file_upload_flag`, `segment_cron_status`.
</details>

---

### 8.2 `s15_v3_segment_user_log` → `segment_members`

> **Filter:** Only rows where `is_eligible = 1` are migrated.

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `segment_id` | `segment_id` | Lookup `segments.id` via `migrated_segment_id` |
| `uid` | `user_id` | Lookup `users.id` via `uid → member_id` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 8.3 `s15_v3_issuance_segment_purchase_qualified` → `segment_qualified_transactions`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `segment_id` | `segment_id` | Lookup `segments.id` via `migrated_segment_id` |
| `user_id` | `user_id` | Lookup `users.id` via `member_id` |
| `milestone_id` | `campaign_milestone_id` | Lookup `campaign_milestones.id` via `migrated_milestone_id` |
| `individual_validity_id` | `campaign_user_activation_id` | Lookup `campaign_user_activations.id` |
| `segment_mapping_id` | `segment_rule_id` | Lookup `segment_rules.id` |
| `create_date` | `created_at` | |

---

### 8.4 `s15_v3_segment_and_or_mapping` → `segment_rule_groups`

> **Note:** Count-only comparison. Each top-level row (`section_and_or_group = 0`) in `s15_v3_segment_and_or_mapping` produces one PG `segment_rule_group`. Record-level comparison is not supported — PG `id` is an auto-generated UUID with no migrated ID stored back in MySQL.

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `segment_id` | `segment_id` | Lookup `segments.id` via `migrated_segment_id` |
| *(derived)* | `order_index` | Derived from row ordering in `s15_v3_segment_and_or_mapping` |
| `db_add_date` | `created_at` | |
| `db_update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped MySQL filter:** Only rows where `section_and_or_group = 0` produce PG rows.
**Unmapped PostgreSQL:** `id` (auto-generated UUID).
</details>

---

## 9. Rewards

### Table Mapping Summary

| MySQL Table | PostgreSQL Table | Notes |
|---|---|---|
| `s15_v3_reward_detail` | `rewards` | Core reward fields |
| `s15_v3_reward_detail` | `reward_coupon_settings` | `reward_code_*` fields |
| `s15_v3_reward_detail` | `reward_expiration_settings` | `reward_expiration_*` fields |
| `s15_v3_reward_detail` | `reward_auto_redemption_rules` | `automate_reward_*` fields |
| `s15_v3_reward_detail` | `reward_eligibility_rules` | `credit_required_*` fields |
| `s15_v3_reward_detail` | `reward_redemption_rules` | `redemption_*` fields |
| `s15_v3_reward_detail` | `reward_notification_settings` | `coupon_alert_*` fields |
| `s15_v3_reward_images` | `reward_images` | |
| `s15_v3_reward_category` | `reward_categories` | |
| `s15_v3_reward_issuances` | `reward_issuance_settings` | |
| `s15_v3_reward_detail` | `reward_eligibility_segment_mappings` | Derived from `reward_segment_ids` CSV |
| `s15_v3_reward_codes` | `reward_codes` | |
| `s15_v3_reward_code_expiration_details` | `reward_code_expirations` | |

---

### 9.1 `s15_v3_reward_detail` → `rewards`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `reward_id` | `migrated_reward_id` | |
| `id` (varchar 50) | `id` | |
| `reward_name` | `name` | |
| `reward_description` | `description` | |
| `reward_terms` | `terms` | |
| `category_id` | `reward_category_id` | Lookup `reward_categories.id` via `migrated_category_id` |
| `reward_type` | `type` | |
| `reward_type_updated` | `redemption_mode` | `enum: {1:1, 2:2, default:3}` |
| `deduct_amount` | `discount_amount` | |
| `reward_currency` | `currency` | |
| `status` | `is_active` | `enum: {0:false, 1:true}` |
| `default_reward_code` | `default_code` | |
| `product_id` | `product_id` | |
| `credit_required` | `points_required` | |
| `reward_image_url` | `image_url` | |
| `reward_url` | `reward_url` | |
| `video_url` | `video_url` | |
| `approval_status` | `approval_status` | `enum: {1:draft, 2:scheduled, 3:live, 4:archived}` |
| `limited_availability` | `limited_availability_enabled` | `enum: {0:false, 1:true}` |
| `availability_start_date` | `limited_availability_start_date` | |
| `availability_end_date` | `limited_availability_end_date` | |
| `reward_sequence` | `sequence` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 9.2 `s15_v3_reward_detail` → `reward_coupon_settings`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `reward_id` *(via `migrated_reward_id`)* | `reward_id` | |
| `reward_code_expiration_flag` | `expiration_enabled` | `enum: {0:false, 1:true}` |
| `reward_code_expiration_start_from` | `expiration_start_from` | `enum: {0:CODE_UPLOAD, 1:CODE_CLAIM}` |
| `reward_code_expiration_type` | `expiration_type` | |
| `reward_code_expiration_days` | `expires_after_days` | |
| `reward_code_expiration_date` | `fixed_expiration_date` | |
| `reward_code_limit` | `usage_limit` | |
| `reward_code_period` | `usage_limit_period` | |
| `reward_code_period_start_date` | `usage_limit_start_date` | |
| `reward_code_period_end_date` | `usage_limit_end_date` | |

---

### 9.3 `s15_v3_reward_detail` → `reward_expiration_settings`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `reward_id` *(via `migrated_reward_id`)* | `reward_id` | |
| `reward_expiration_type` | `expiration_type` | |
| `reward_expiration_days` | `days_after_claim` | |
| `reward_expiration_start_date` | `expiration_start_at` | |
| `reward_expiration_end_date` | `expiration_end_at` | |

---

### 9.4 `s15_v3_reward_detail` → `reward_auto_redemption_rules`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `reward_id` *(via `migrated_reward_id`)* | `reward_id` | |
| `automate_reward_flag` | `enabled` | `enum: {0:false, 1:true}` |
| `automate_reward_type` | `basis` | `enum: {1:LIFETIME_POINTS, 2:AVAILABLE_POINT, 3:PURCHASE_AMOUNT}` |
| `automate_reward_claim_flag` | `threshold_type` | `enum: {0:ROLLING, 1:FIXED}` |
| `automate_reward_credit_required` | `threshold_value` | |
| `automate_reward_claim_limit` | `limit` | |
| `automate_reward_claim_reason` | `reason` | |
| `time_base_issuance_flag` | `date_enabled` | `enum: {0:false, 1:true}` |
| `time_base_issuance_date` | `date` | |

---

### 9.5 `s15_v3_reward_detail` → `reward_eligibility_rules`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `reward_id` *(via `migrated_reward_id`)* | `reward_id` | |
| `reward_eligibility_threshold` | `threshold_mode` | `enum: {0:FIXED, 1:RECURRING}` |
| `credit_required_flag` | `threshold_basis` | `enum: {0:LIFETIME_POINTS, 1:PURCHASE_AMOUNT, 2:SEGMENT, 3:TIER}` |
| `credit_required_flag_value` | `threshold_value` | |

---

### 9.6 `s15_v3_reward_detail` → `reward_redemption_rules`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `reward_id` *(via `migrated_reward_id`)* | `reward_id` | |
| `reward_eligible_year` | `eligibility_year` | |
| `reward_eligible_flag` | `eligibility_period_unit` | `enum: {1:MONTHLY, 2:QUARTERLY, 3:HALF_YEARLY, 4:ANNUALLY}` |
| `reward_eligible_value` | `eligibility_period_value` | |
| `redemption_limit` | `limit_per_member` | |
| `reward_claim_limit` | `limit_total` | |
| `redemption_limit_duration` | `member_limit_period` | |

---

### 9.7 `s15_v3_reward_detail` → `reward_notification_settings`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `reward_id` *(via `migrated_reward_id`)* | `reward_id` | |
| `coupon_alert_count` | `low_inventory_threshold` | |
| `coupon_alert_email` | `alert_email` | |
| `coupon_alert_email_subject` | `alert_email_subject` | |
| `coupon_alert_email_body` | `alert_email_body` | |

---

### 9.8 `s15_v3_reward_images` → `reward_images`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `reward_id` | `reward_id` | Lookup `rewards.id` via `migrated_reward_id` |
| `image_url` | `image_url` | |
| `default_thumb_image` | `is_default_thumb_image` | `enum: {0:false, 1:true}` |
| `image_attribute_type` | `attribute_type` | |
| `image_attribute_height` | `attribute_height` | |
| `image_attribute_width` | `attribute_width` | |
| `image_attribute_background_color` | `attribute_background_color` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 9.9 `s15_v3_reward_category` → `reward_categories`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `id` | `migrated_category_id` | |
| `category_name` | `name` | |
| `reward_category_status` | `is_active` | `enum: {0:false, 1:true}` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 9.10 `s15_v3_reward_codes` → `reward_codes`

> **Sources:** `s15_v3_reward_codes` (individual codes) UNION `s15_v3_reward_detail.default_reward_code`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `reward_id` | `reward_id` | Lookup `rewards.id` via `migrated_reward_id` |
| `reward_code` | `code` | From `s15_v3_reward_codes` |
| `default_reward_code` | `code` | From `s15_v3_reward_detail` (UNION ALL) |
| `reward_status` | `is_available` | `enum: {0:false, 1:true}` |
| *(default)* | `is_default` | `false` for codes rows; `true` for `default_reward_code` rows |

---

### 9.11 `s15_v3_reward_code_expiration_details` → `reward_code_expirations`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `user_id` | `user_id` | Lookup `users.id` via `member_id` |
| `reward_code_id` | `reward_code_id` | Lookup `reward_codes.id` via `reward_code` value |
| `reward_code_expiration_date` | `expiry_date` | |
| `process_status` | `is_expired` | `enum: {0:false, 1:true}` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 9.12 `s15_v3_reward_issuances` → `reward_issuance_settings`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `reward_id` | `reward_id` | Lookup `rewards.id` via `migrated_reward_id` |
| `member_event_type` | `member_event_type` | `enum: {0:null, 1:BIRTHDAY, 2:LOYALTY_ANNIVERSARY}` |
| `issue_on` | `issue_on` | `enum: {0:null, 1:EVENT_DAY, 2:DAY_OF_EVENT_MONTH, 3:BEFORE_DAY}` |
| `event_days` | `event_days` | |
| `created_at` | `created_at` | |
| `updated_at` | `updated_at` | |

---

### 9.13 `s15_v3_reward_detail` → `reward_eligibility_segment_mappings`

> **Derived table:** Each non-empty `reward_segment_ids` field in `s15_v3_reward_detail` is a comma-separated list. One PG row is produced per segment ID entry. Count is validated via `SUM(LENGTH(reward_segment_ids) - LENGTH(REPLACE(reward_segment_ids, ',', '')) + 1)` across all applicable rows.

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| *(auto-generated)* | `id` | Auto-generated UUID |
| `reward_id` *(via `reward_eligibility_rules.reward_id`)* | `reward_eligibility_rule_id` | FK to `reward_eligibility_rules` |
| `reward_segment_ids` *(split CSV)* | `segment_id` | Each comma-separated value becomes one row |
| *(no MySQL source)* | `created_at` | |
| *(no MySQL source)* | `updated_at` | |

---

## 10. Transactions & Points

### Table Mapping Summary

| MySQL Table | PostgreSQL Table | Notes |
|---|---|---|
| `s15_v3_transaction` | `transactions` | Range-partitioned by `created_at` |
| `s15_v3_order_detail` + `s15_v3_product_detail` | `transaction_items` | Range-partitioned by `transaction_created_at` |
| `s15_v3_order_info` | `point_allocations` | |
| `s15_v3_order_delivery` | `item_shippings` | |
| `s15_v3_order_points_differentiation_log` | `point_allocation_change_logs` | |
| `s15_v3_exclude_include_product_detail` | `item_rules` | |
| `s15_v3_points_balance_summary` | `point_balances` | |
| `s15_v3_exclude_include_product_detail` | `items` | Filter: `points_ratio_type IN (0,1)` AND `product_id IS NOT NULL` |
| `s15_v3_exclude_include_product_detail` | `categories` | Filter: `points_ratio_type = 2` AND `product_category_id IS NOT NULL` |
| `s15_v3_exclude_include_product_detail` | `item_category_mappings` | Filter: `product_id IS NOT NULL` AND `product_category_id IS NOT NULL` |

---

### 10.1 `s15_v3_transaction` → `transactions`

> **Note:** `transactions` is a range-partitioned table in PostgreSQL, partitioned by `created_at`.

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `id` (varchar 50) | `id` | UUID preserved |
| `site_id` | `site_id` | |
| `action_id` | `action_id` | |
| `user_id` | `user_id` | Lookup `users.id` via `uid → member_id` |
| `group_id` | `group_id` | Lookup `groups.id` via `migrated_group_id` |
| `credit` | `credit` | |
| `debit` | `debit` | |
| `group_credit` | `group_credit` | `COALESCE(group_credit, 0)` |
| `group_debit` | `group_debit` | `COALESCE(group_debit, 0)` |
| `transaction_id` | `old_transaction_id` | Legacy integer PK |
| `expire_date` | `expiry_date` | |
| `create_date` | `order_date` | **Note:** MySQL `create_date` maps to `order_date` in PG |
| `update_date` | `updated_at` | Fallback to `created_at` if NULL |

<details>
<summary>Unmapped columns</summary>

**Unmapped PostgreSQL:** `partner_code`, `order_id`, `order_total_spend`, `shipped_order_total`, `order_discount_amount`, `order_total_points`, `points_on_hold`, `order_status`, `order_ship_date`, `reward_id`, `reward_code`, `reward_status`, `reward_used_at`, `is_large_value_order`, `reason`, `admin_email`, `source`, `coupon`, `store_id`, `point_type`, `custom_id`, `sender_type`, `sender_id`, `receiver_type`, `receiver_id`, `country_id`, `action_attributes` (JSONB), `issuance_order_attributes` (JSONB), `created_at`, `is_deleted`, `deleted_at`.
</details>

---

### 10.2 `s15_v3_order_detail` + `s15_v3_product_detail` → `transaction_items`

> **Note:** `transaction_items` is range-partitioned by `transaction_created_at`. Primary source: `s15_v3_order_detail`; `s15_v3_product_detail` provides product metadata.

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `s15_v3_order_detail.site_id` | `site_id` | |
| `s15_v3_order_detail.transaction_id` | `transaction_id` | Lookup `transactions.id` via `old_transaction_id` |
| `s15_v3_order_detail.transaction_uuid` | `transaction_created_at` | Used to resolve transaction partition key |
| `s15_v3_order_detail.product_id` | `item_id` | |
| `s15_v3_order_detail.product_qty` | `item_quantity` | |
| `s15_v3_order_detail.unit_price` | `item_price` | |
| `s15_v3_order_detail.product_name` | `item_name` | |
| `s15_v3_order_detail.product_description` | `item_description` | |
| `s15_v3_order_detail.product_category_id` | `category_code` | |
| `s15_v3_order_detail.product_category_name` | `category_name` | |
| `s15_v3_order_detail.product_discount` | `item_discount_amount` | |
| `s15_v3_order_detail.status` | `status` | `enum: {0:hold, 1:ship, 2:released, 4:return, 8:cancel}` |
| `s15_v3_order_detail.ship_qty` | `ship_qty` | |
| `s15_v3_order_detail.return_qty` | `return_qty` | |
| `s15_v3_order_detail.cancel_qty` | `cancel_qty` | |
| `s15_v3_order_detail.points_ratio` | `multiplier` | |
| `s15_v3_order_detail.coupon` | `item_coupon_code` | |
| `s15_v3_order_detail.release_date` | `item_ship_date` | |
| `s15_v3_order_detail.create_date` | `created_at` | |
| `s15_v3_order_detail.update_date` | `updated_at` | |
| `s15_v3_product_detail.upc` | `item_code` | |
| `s15_v3_product_detail.mpn` | `secondary_id` | |
| `s15_v3_product_detail.brand` + others | `custom_attributes` | JSONB — see Appendix |

---

### 10.3 `s15_v3_order_info` → `point_allocations`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `order_detail_id` | `transaction_item_id` | Lookup `transaction_items.id` |
| `transaction_uuid` | `transaction_created_at` | Used for partition key resolution |
| `campaign_id` | `campaign_id` | Lookup `campaigns.id` via `migrated_campaign_id` |
| `milestone_id` | `campaign_milestone_id` | Lookup `campaign_milestones.id` via `migrated_milestone_id` |
| `campaign_ratio` | `milestone_multiplier` | |
| `tier_ratio` | `tier_multiplier` | |
| `action_ratio` | `action_multiplier` | |
| `exclude_include_ratio` | `item_multiplier` | |
| `purchase_point_type` | `point_type` | `enum: {1:ratio, 2:flat}` |
| `base_points` | `base_points` | |
| `campaign_points` | `campaign_points` | |
| `total_points` | `total_points` | |
| `flat_bonus_points` | `item_flat_points` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 10.4 `s15_v3_order_delivery` → `item_shippings`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `order_detail_id` | `transaction_item_id` | Lookup `transaction_items.id` |
| `ship_qty` | `ship_qty` | |
| `return_qty` | `return_qty` | |
| `release_date` | `release_date` | |
| `status` | `status` | `enum: {0:hold, 1:return, 4:ship, 8:released}` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 10.5 `s15_v3_order_points_differentiation_log` → `point_allocation_change_logs`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `order_detail_id` | `transaction_item_id` | Lookup `transaction_items.id` |
| `order_discount` | `discount` | |
| `base_points` | `base_points` | |
| `campaign_points` | `campaign_points` | |
| `total_points` | `total_points` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 10.6 `s15_v3_exclude_include_product_detail` → `item_rules`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `product_id` | `entity_id` | When `points_ratio_type` in (0,1) → product rule |
| `product_category_id` | `entity_id` | When `points_ratio_type = 2` → category rule |
| `product_name` | `name` | For product rules |
| `product_category_name` | `name` | For category rules |
| `points_ratio_type` | `entity_type` | `enum: {0:product, 1:product, 2:category}` |
| `status` | `eligible` | `enum: {0:false, 1:true}` (product-level) |
| `category_points_type` | `eligible` | For category rules |
| `points_ratio` | `multiplier` | For product rules |
| `category_points_ratio` | `multiplier` | For category rules |
| `bonus_points` | `flat_points` | For product rules |
| `cat_flat_points_value` | `flat_points` | For category rules |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 10.7 `s15_v3_points_balance_summary` → `point_balances`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `entity_type` | `entity_type` | `user` or `group` |
| `entity_id` | `entity_id` | For `user`: lookup `users.id` via `uid → member_id`; for `group`: lookup `groups.id` via `migrated_group_id` |
| `current_balance` | `current_balance` | |
| `available_balance` | `available_balance` | |
| `lifetime_credit` | `lifetime_credit` | |
| `lifetime_debit` | `lifetime_debit` | |
| `lifetime_expired` | `lifetime_expired` | |
| `hold_points` | `hold_points` | |
| `redeem_points` | `redeem_points` | |
| `ytd_credit` | `ytd_credit` | |
| `ytd_debit` | `ytd_debit` | |
| `ytd_year` | `ytd_year` | |
| `total_spend` | `total_spend` | |
| `point_to_expire` | `point_to_expire` | |
| `non_expiry_points` | `non_expiry_points` | |
| `total_purchase_hold_points` | `total_purchase_hold_points` | |
| `used_redemption_points` | `used_redemption_points` | |
| `available_redemption_points` | `available_redemption_points` | |
| `point_expiration_date` | `point_expiration_date` | |
| `last_transaction_at` | `last_transaction_at` | |
| `created_at` | `created_at` | |
| `updated_at` | `updated_at` | |

---

### 10.8 `s15_v3_exclude_include_product_detail` → `items`

> **Filter:** Only rows where `points_ratio_type IN (0, 1)` and `product_id IS NOT NULL`.

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `product_id` | `item_id` | |
| `product_name` | `name` | |
| `upc` | `attributes` | Stored as key `upc` inside `attributes` JSONB |
| `mpn` | `attributes` | Stored as key `mpn` inside `attributes` JSONB |
| `gtin` | `attributes` | Stored as key `gtin` inside `attributes` JSONB |
| `brand_id` | `attributes` | Stored as key `brand_id` inside `attributes` JSONB |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped PostgreSQL:** `id` (auto-generated UUID).
</details>

---

### 10.9 `s15_v3_exclude_include_product_detail` → `categories`

> **Filter:** Only rows where `points_ratio_type = 2` and `product_category_id IS NOT NULL`.

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `product_category_id` | `category_id` | |
| `product_category_name` | `name` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped PostgreSQL:** `id` (auto-generated UUID).
</details>

---

### 10.10 `s15_v3_exclude_include_product_detail` → `item_category_mappings`

> **Filter:** Only rows where `product_id IS NOT NULL` and `product_category_id IS NOT NULL`.

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `product_id` | `item_id` | |
| `product_category_id` | `category_id` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

<details>
<summary>Unmapped columns</summary>

**Unmapped PostgreSQL:** `id` (auto-generated UUID).
</details>

---

## 11. Stores

### Table Mapping Summary

| MySQL Table(s) | PostgreSQL Table | Notes |
|---|---|---|
| `sa_store_details` | `stores` | |
| `s15_v3_store_attribute` + `s15_v3_store_attribute_values` | `stores` | `custom_attributes` JSONB column |

---

### 11.1 `sa_store_details` → `stores`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `store_id` | `store_id` | |
| `store_name` | `name` | |
| `street` | `street_address` | |
| `level_1` through `level_10` | `custom_attributes` | JSONB: `{"level_1":"...", ..., "level_10":"..."}` |
| `db_add_date` | `created_at` | |
| `db_update_date` | `updated_at` | |
| *(from `s15_v3_store_attribute` + `s15_v3_store_attribute_values`)* | `custom_attributes` | Merged JSONB — see Appendix |

---

## 12. Programs & Timezones

### Table Mapping Summary

| MySQL Table | PostgreSQL Table |
|---|---|
| `s15_v3_programs` | `programs` |
| `timezone_list` | `timezones` |

---

### 12.1 `s15_v3_programs` → `programs`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `id` (varchar 40) | `id` | |
| `program_name` | `name` | |
| `program_description` | `description` | |
| `program_status` | `status` | `enum: {1:scheduled, 2:live, 3:archived, default:draft}` |
| `program_start_date` | `start_date` | |
| `program_bypass_condition` | `bypass_condition` | `enum: {0:false, 1:true}` |
| `tier_notification` | `notification_enabled` | `enum: {0:false, 1:true}` |
| `created_at` | `created_at` | |
| `updated_at` | `updated_at` | |

---

### 12.2 `timezone_list` → `timezones`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `timezone` | `tz_code` | |
| `timezone_name` | `name` | |

---

## 13. Localisation

### Table Mapping Summary

| MySQL Table | PostgreSQL Table |
|---|---|
| `s15_v3_supported_multi_language` | `locales` |
| `s15_v3_text_translation_multi_language` | `translations` |
| `s15_v3_syncronize_text_multi_language` | `translation_contents` |
| `s15_v3_multi_language_synchronize_data` | `translation_sync_jobs` |
| `s15_v3_language_master` | `language_masters` |

---

### 13.1 `s15_v3_supported_multi_language` → `locales`

**Sources:** `s15_v3_supported_multi_language` (primary), `s15_v3_language_master` (joined for `code` / `name`)

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `support_lang_id` | `migrated_language_id` | |
| `site_id` | `site_id` | |
| `custom_language_code` | `standard_code_alias` | |
| `s15_v3_language_master.language_code` | `standard_code` | Joined from `s15_v3_language_master` |
| `s15_v3_language_master.language_name` | `name` | Joined from `s15_v3_language_master` |
| `s15_v3_language_master.is_rtl` | `is_rtl` | Joined from `s15_v3_language_master` |
| `is_active` | `is_active` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 13.2 `s15_v3_text_translation_multi_language` → `translations`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `sync_lang_id` | `translation_content_id` | Lookup `translation_contents.id` via `migrated_content_id` |
| `support_lang_id` | `locale_id` | Lookup `locales.id` via `migrated_language_id` |
| `translated_value` | `translated_content_text` | |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 13.3 `s15_v3_syncronize_text_multi_language` → `translation_contents`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `sync_lang_id` | `migrated_content_id` | |
| `english_value` | `content_text` | |
| `field_type` | `content_type` | Default `TEXT` when `field_type = 0` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 13.4 `s15_v3_multi_language_synchronize_data` → `translation_sync_jobs`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `site_id` | `site_id` | |
| `module` | `module` | `integer → varchar` |
| `sub_module` | `sub_module` | `integer → varchar` |
| `is_active` | `source_record_state` | `enum: {0:ALL, 1:ACTIVE}` |
| `status` | `status` | `enum: {0:REQUESTED, 1:STARTED, 2:COMPLETED, 3:ERROR}` |
| `create_date` | `created_at` | |
| `update_date` | `updated_at` | |

---

### 13.5 `s15_v3_language_master` → `language_masters`

| MySQL Column | PostgreSQL Column | Notes |
|---|---|---|
| `lang_master_id` | `id` | |
| `language_name` | `name` | |
| `language_code` | `code` | |
| `is_rtl` | `is_rtl` | `enum: {0:false, 1:true}` |
| `create_date` | `created_at` | |

---

## Unmapped MySQL Tables

The following MySQL tables have **no identified PostgreSQL destination** and have not been migrated:

| MySQL Table | Notes |
|---|---|
| `s15_v3_action_hold_point` | Hold points logic — no direct PG equivalent |
| `s15_v3_actiongroup_user_track` | User tracking for action groups |
| `s15_v3_bucket_master` | Multi-bucket configuration |
| `s15_v3_multibucket_actiongroup_detail` | Multi-bucket action group associations |
| `s15_v3_multibuckets_hold_transaction` | Hold transactions for multi-buckets |
| `s15_v3_multibuckets_transaction` | Multi-bucket transactions |
| `s15_v3_reward_redemption_attributes` | EAV attribute definitions for redemptions |
| `s15_v3_reward_redemption_attribute_values` | EAV attribute values for redemptions |
| `s15_v3_segment_and_or` | Segment rule AND/OR logic |
| `s15_v3_segment_users_delete_log` | Deleted segment member audit log |
| `s15_v3_user_opt_in_out_history` | User opt-in/out history |
| `site_grouping` | Site grouping configuration |

---

## PostgreSQL Tables with No MySQL Source

The following PostgreSQL tables are **new in the PG schema** and have no direct MySQL source table:

| PostgreSQL Table | Notes |
|---|---|
| `segment_rules` | New PG structure for segment rule conditions (sourced from denormalized columns in `s15_v3_segment`) |
| `segment_entity_sets` | New PG structure for segment entity set definitions |
| `segment_entity_set_items` | New PG structure for items in entity sets |

---

## Appendix: EAV → JSONB Flattening Pattern

Several MySQL tables store custom attributes using an **Entity-Attribute-Value (EAV)** pattern via paired `_attribute` and `_attribute_values` tables. During migration these are flattened into a single `custom_attributes` **JSONB** column on the corresponding PostgreSQL table.

### EAV Table Pairs → Target JSONB Column

| MySQL Source Tables | PostgreSQL Target | JSONB Column |
|---|---|---|
| `s15_v3_user_attribute` + `s15_v3_user_attribute_values` | `users` | `custom_attributes` |
| `s15_v3_badge_attributes` + `s15_v3_badge_attribute_values` | `badges` | `custom_attributes` |
| `s15_v3_store_attribute` + `s15_v3_store_attribute_values` | `stores` | `custom_attributes` |
| `s15_v3_reward_redemption_attributes` + `s15_v3_reward_redemption_attribute_values` | *(reward redemption / transaction issuance attributes)* | `issuance_order_attributes` |

### Additional JSONB Mappings

| MySQL Source | PostgreSQL Target | JSONB Column | Pattern |
|---|---|---|---|
| `s15_v3_badge_images` | `badges` | `images` | `[{"image_url":"...", "is_default":true, ...}, ...]` |
| `sa_store_details.level_1..level_10` | `stores` | `custom_attributes` | `{"level_1":"val", ..., "level_10":"val"}` |

### Flattening Algorithm

1. For each entity (user / badge / store / reward), collect all attribute key-value pairs from the `_attribute_values` table, joined with `_attribute` for key names.
2. Build a JSON object: `{"key1": "value1", "key2": "value2", ...}`.
3. Store the resulting JSON object in the target JSONB column.
4. **Multitemplate behaviour:** EAV values are resolved per `site_id`, with multitemplate fallback handled at the sites level — see [Global Notes: multitemplate_id](#multitemplate_id).
