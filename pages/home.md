---
name: Home
assetId: 575f3fed-96a5-4371-b0bd-9a600943a002
type: page
---

# Marketing Dashboard Home Page

{% range_calendar id="date_filter" /%}

```sql scorecards
SELECT
  count(`Lead Status`) as total_leads,
  countIf(`Lead Status` = 'Dead Lead') as dead_leads,
  countIf(`Offer Date` > toDate('1970-01-01')) as offers_made,
  countIf(`Under Contract Date` > toDate('1970-01-01')) as contracts,
  countIf(`Under Contract Date` > toDate('1970-01-01')) as deals,
  `Lead Created Date` as Date
FROM resimpli_export_raw
GROUP BY Date
```

{% big_value data="scorecards" value="sum(total_leads)" title="Total Leads" fmt="num0" date_range={
  range="{{date_filter}}"
  date="Date"
} /%}

{% big_value data="scorecards" value="sum(dead_leads)" title="Dead Leads" fmt="num0" date_range={
  date="Date"
  range="{{date_filter}}"
}/%}

{% big_value data="scorecards" value="sum(offers_made)" title="Offers Made" fmt="num0" date_range={
  range="{{date_filter}}"
  date="Date"
} /%}

{% big_value data="scorecards" value="sum(contracts)" title="Contracts" fmt="num0" date_range={
  range="{{date_filter}}"
  date="Date"
} /%}

{% big_value data="scorecards" value="sum(deals)" title="Deals" fmt="num0" info="Placeholder: using Contracts as proxy until Deals data is available" date_range={
  range="{{date_filter}}"
  date="Date"
} /%}

## New vs. Old Cohort

```sql cohort_data
SELECT
  toDate(`Under Contract Date`) as contract_month,
  IF(dateDiff('day', `Lead Created Date`, `Under Contract Date`) <= 60, 'New Lead Deal', 'Follow-up Deal') as cohort,
  count(*) as deals,
  toDate(`Lead Created Date`) as Date
FROM resimpli_export_raw
WHERE `Under Contract Date` > toDate('1970-01-01')
  AND `Lead Created Date` > toDate('1970-01-01')
GROUP BY contract_month, cohort, Date
ORDER BY contract_month, cohort
```

{% bar_chart
  data="cohort_data"
  x="contract_month"
  y="sum(deals)"
  series="cohort"
  stacked=true
  title="New Lead Deals vs. Follow-up Deals"
  subtitle="New = contract within 60 days of lead creation; Follow-up = more than 60 days"
  x_fmt="mmm yyyy"
  date_range={
    date="Date"
    range={{date_filter}}
  }
  fmt="num0"
  y_fmt="num0"
  date_grain="week"
  chart_options={
    series_colors={
      "New Lead Deal"="#22c55e"
      "Follow-up Deal"="#f59e0b"
    }
  }
/%}

## Channel Logic

```sql channel_logic
SELECT
  `Lead Source` as channel,
  count(*) as leads,
  countIf(`Under Contract Date` > toDate('1970-01-01')) as contracts,
  countIf(`Under Contract Date` > toDate('1970-01-01')) as closed_deals,
  countIf(`Under Contract Date` > toDate('1970-01-01')) / count(*) as conversion_rate,
  toDate(`Lead Created Date`) as Date
FROM resimpli_export_raw
WHERE `Lead Source` != ''
GROUP BY channel, Date
ORDER BY leads DESC
```

{% table data="channel_logic" date_range={
  date="Date"
  range={{date_filter}}
} %}
  {% dimension value="channel" title="Channel" /%}
  {% measure value="sum(leads)" title="Leads" fmt="num0" /%}
  {% measure value="sum(contracts)" title="Contracts" fmt="num0" /%}
  {% measure value="sum(closed_deals)" title="Closed Deals" fmt="num0" info="Placeholder: using Contracts as proxy until Deals data is available" /%}
  {% measure value="sum(conversion_rate)" title="Conversion Rate" fmt="pct1" /%}
{% /table %}

## Lead Funnel

```sql funnel_data
SELECT stage, stage_order, count, Date FROM (
  SELECT 'Total Leads' as stage, 1 as stage_order, count(*) as count, toDate(`Lead Created Date`) as Date
  FROM resimpli_export_raw
  GROUP BY Date
  UNION ALL
  SELECT 'Contact Made', 2, countIf(`Lead Status` IN ('Contact Made','Appointments Set','Cancelled Appointments','Offers Made','Under Contract')), toDate(`Lead Created Date`)
  FROM resimpli_export_raw
  GROUP BY toDate(`Lead Created Date`)
  UNION ALL
  SELECT 'Appointments', 3, countIf(`Appointment Date` > toDate('1970-01-01')), toDate(`Lead Created Date`)
  FROM resimpli_export_raw
  GROUP BY toDate(`Lead Created Date`)
  UNION ALL
  SELECT 'Offers Made', 4, countIf(`Offer Date` > toDate('1970-01-01')), toDate(`Lead Created Date`)
  FROM resimpli_export_raw
  GROUP BY toDate(`Lead Created Date`)
  UNION ALL
  SELECT 'Contracts', 5, countIf(`Under Contract Date` > toDate('1970-01-01')), toDate(`Lead Created Date`)
  FROM resimpli_export_raw
  GROUP BY toDate(`Lead Created Date`)
) ORDER BY stage_order
```

{% funnel_chart
  data="funnel_data"
  category="stage"
  value="sum(count)"
  title="Overall Lead Funnel"
  show_percent=true
  align="center"
  order="stage_order"
  date_range={
    date="Date"
    range={{date_filter}}
  }
/%}

### Funnel by Channel

```sql funnel_by_channel
SELECT channel, stage, stage_order, count, Date FROM (
  SELECT `Lead Source` as channel, 'Total Leads' as stage, 1 as stage_order, count(*) as count, toDate(`Lead Created Date`) as Date
  FROM resimpli_export_raw
  WHERE `Lead Source` != ''
  GROUP BY channel, Date
  UNION ALL
  SELECT `Lead Source`, 'Contact Made', 2, countIf(`Lead Status` IN ('Contact Made','Appointments Set','Cancelled Appointments','Offers Made','Under Contract')), toDate(`Lead Created Date`)
  FROM resimpli_export_raw
  WHERE `Lead Source` != ''
  GROUP BY `Lead Source`, toDate(`Lead Created Date`)
  UNION ALL
  SELECT `Lead Source`, 'Appointments', 3, countIf(`Appointment Date` > toDate('1970-01-01')), toDate(`Lead Created Date`)
  FROM resimpli_export_raw
  WHERE `Lead Source` != ''
  GROUP BY `Lead Source`, toDate(`Lead Created Date`)
  UNION ALL
  SELECT `Lead Source`, 'Offers Made', 4, countIf(`Offer Date` > toDate('1970-01-01')), toDate(`Lead Created Date`)
  FROM resimpli_export_raw
  WHERE `Lead Source` != ''
  GROUP BY `Lead Source`, toDate(`Lead Created Date`)
  UNION ALL
  SELECT `Lead Source`, 'Contracts', 5, countIf(`Under Contract Date` > toDate('1970-01-01')), toDate(`Lead Created Date`)
  FROM resimpli_export_raw
  WHERE `Lead Source` != ''
  GROUP BY `Lead Source`, toDate(`Lead Created Date`)
) ORDER BY stage_order, channel
```

{% bar_chart
  data="funnel_by_channel"
  x="stage"
  y="sum(count)"
  series="channel"
  stacked=true
  title="Funnel Breakdown by Channel"
  subtitle="Each stage shows the contribution from each lead source"
  order="min(stage_order)"
  date_range={
    date="Date"
    range={{date_filter}}
  }
/%}