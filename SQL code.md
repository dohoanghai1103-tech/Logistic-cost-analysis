# Logistic-cost-analysis - SQL code 
-- 1.1 KPI tổng: Volume, Cost, Cost/Shipment, On-time Rate theo half_year
SELECT
    p.half_year,
    SUM(p.shipment_volume)                                  AS total_volume,
    SUM(c.total_delivery_cost)                               AS total_cost,
    SUM(c.total_delivery_cost) * 1.0 / SUM(p.shipment_volume) AS cost_per_shipment,
    SUM(p.on_time_deliveries) * 1.0 / SUM(p.shipment_volume)  AS on_time_rate
FROM fact_delivery_performance p
JOIN fact_delivery_cost c
    ON p.date = c.date AND p.hub_id = c.hub_id
   AND p.service_type = c.service_type AND p.route_type = c.route_type
   AND p.payment_type = c.payment_type AND p.size_type = c.size_type
GROUP BY p.half_year;
-- 1.2 Growth % giữa 2 kỳ (pivot bằng CASE WHEN)
SELECT
    SUM(CASE WHEN half_year = 'H1_2025' THEN total_volume END) * 1.0
    / SUM(CASE WHEN half_year = 'H2_2024' THEN total_volume END) - 1 AS volume_growth,
    SUM(CASE WHEN half_year = 'H1_2025' THEN total_cost END) * 1.0
    / SUM(CASE WHEN half_year = 'H2_2024' THEN total_cost END) - 1  AS cost_growth
FROM ( /* subquery giống 1.1 */ ) t;
-- 2.1 Verify 5 cấu phần cộng = total_delivery_cost (0 dòng lệch)
SELECT COUNT(*) AS mismatch_rows
FROM fact_delivery_cost
WHERE ABS(labor_cost + transportation_cost + hub_handling_cost
          + failed_redelivery_cost + outsourcing_cost - total_delivery_cost) > 1;
          -- 2.2 Pivot 2 kỳ thành cột, tính growth% và % contribution mỗi cấu phần
WITH agg AS (
    SELECT half_year,
           SUM(labor_cost) AS labor, SUM(transportation_cost) AS transport,
           SUM(hub_handling_cost) AS hub_handling,
           SUM(failed_redelivery_cost) AS failed_redelivery,
           SUM(outsourcing_cost) AS outsourcing
    FROM fact_delivery_cost GROUP BY half_year
),
unpivot AS (
    SELECT 'Labor' AS component, MAX(CASE WHEN half_year='H2_2024' THEN labor END) AS h2,
           MAX(CASE WHEN half_year='H1_2025' THEN labor END) AS h1 FROM agg
    UNION ALL
    SELECT 'Transportation', MAX(CASE WHEN half_year='H2_2024' THEN transport END),
           MAX(CASE WHEN half_year='H1_2025' THEN transport END) FROM agg
    UNION ALL
    SELECT 'Hub Handling', MAX(CASE WHEN half_year='H2_2024' THEN hub_handling END),
           MAX(CASE WHEN half_year='H1_2025' THEN hub_handling END) FROM agg
    UNION ALL
    SELECT 'Failed/Redelivery', MAX(CASE WHEN half_year='H2_2024' THEN failed_redelivery END),
           MAX(CASE WHEN half_year='H1_2025' THEN failed_redelivery END) FROM agg
    UNION ALL
    SELECT 'Outsourcing', MAX(CASE WHEN half_year='H2_2024' THEN outsourcing END),
           MAX(CASE WHEN half_year='H1_2025' THEN outsourcing END) FROM agg
)
SELECT component, h2, h1,
       (h1 - h2) * 1.0 / h2                              AS growth_pct,
       (h1 - h2) * 1.0 / (SUM(h1-h2) OVER ())             AS pct_of_total_increase
FROM unpivot
ORDER BY pct_of_total_increase DESC;
-- 3.1 Volume Effect vs Price Effect decomposition
WITH agg AS (
    SELECT
        SUM(CASE WHEN half_year='H2_2024' THEN outsourced_shipments END) AS vol_h2,
        SUM(CASE WHEN half_year='H1_2025' THEN outsourced_shipments END) AS vol_h1
    FROM fact_delivery_performance
),
cost_agg AS (
    SELECT
        SUM(CASE WHEN half_year='H2_2024' THEN outsourcing_cost END) AS cost_h2,
        SUM(CASE WHEN half_year='H1_2025' THEN outsourcing_cost END) AS cost_h1
    FROM fact_delivery_cost
)
SELECT
    (vol_h1 - vol_h2) * 1.0 / vol_h2                                       AS volume_growth_pct,
    (cost_h1/vol_h1 - cost_h2/vol_h2) * 1.0 / (cost_h2/vol_h2)              AS price_growth_pct,
    (vol_h1 - vol_h2) * (cost_h2*1.0/vol_h2)                                AS volume_effect_amount,
    (cost_h1 - cost_h2) - (vol_h1 - vol_h2) * (cost_h2*1.0/vol_h2)          AS price_effect_amount
FROM agg, cost_agg;
-- 3.2 Confirm D1: Capacity Utilization, Backlog, Overtime theo half_year
SELECT
    half_year,
    AVG(capacity_utilization)                              AS capacity_utilization,
    SUM(backlog_parcels)                                    AS backlog_parcels,
    SUM(overtime_hours) * 1.0 / SUM(working_hours)          AS overtime_ratio
FROM fact_rider_hub_productivity
GROUP BY half_year;
-- 3.3 Outsourced Share vs Internal Shipments growth
SELECT
    half_year,
    SUM(internal_shipments)  AS internal_shipments,
    SUM(outsourced_shipments) AS outsourced_shipments,
    SUM(outsourced_shipments) * 1.0 / (SUM(outsourced_shipments)+SUM(internal_shipments)) AS outsourced_share
FROM fact_delivery_performance
GROUP BY half_year;
-- 3.4 Drill xuống hub: Capacity Utilization by hub, tìm hub vượt 100%
SELECT
    h.hub_name, h.is_focus_hub,
    SUM(CASE WHEN r.half_year='H1_2025' THEN r.internal_shipments END) * 1.0
    / SUM(CASE WHEN r.half_year='H1_2025' THEN r.hub_capacity END) AS capacity_utilization_h1
FROM fact_rider_hub_productivity r
JOIN dim_hub h ON r.hub_id = h.hub_id
GROUP BY h.hub_name, h.is_focus_hub
ORDER BY capacity_utilization_h1 DESC;
-- 3.5 So sánh nhóm Focus vs Normal: Volume/Capacity/Rider Growth
SELECT
    h.is_focus_hub,
    (SUM(CASE WHEN r.half_year='H1_2025' THEN r.internal_shipments END)*1.0
     / SUM(CASE WHEN r.half_year='H2_2024' THEN r.internal_shipments END)) - 1 AS volume_growth,
    (SUM(CASE WHEN r.half_year='H1_2025' THEN r.hub_capacity END)*1.0
     / SUM(CASE WHEN r.half_year='H2_2024' THEN r.hub_capacity END)) - 1      AS capacity_growth,
    (SUM(CASE WHEN r.half_year='H1_2025' THEN r.rider_count END)*1.0
     / SUM(CASE WHEN r.half_year='H2_2024' THEN r.rider_count END)) - 1      AS rider_growth
FROM fact_rider_hub_productivity r
JOIN dim_hub h ON r.hub_id = h.hub_id
GROUP BY h.is_focus_hub;
-- 3.6 Link sang SLA: On-time rate theo Focus/Normal, và SLA vendor
SELECT
    h.is_focus_hub,
    p.half_year,
    SUM(p.on_time_deliveries) * 1.0 / SUM(p.shipment_volume) AS on_time_rate
FROM fact_delivery_performance p
JOIN dim_hub h ON p.hub_id = h.hub_id
GROUP BY h.is_focus_hub, p.half_year;

-- SLA vendor riêng
SELECT
    half_year,
    SUM(vendor_on_time_deliveries) * 1.0 / SUM(outsourced_shipments) AS vendor_on_time_rate
FROM fact_vendor_performance
GROUP BY half_year;
-- 3.7 Decompose giá vendor: Mix Effect vs Rate Effect
SELECT
    v.vendor_tier,
    fp.half_year,
    SUM(fp.outsourced_shipments) * 1.0
      / SUM(SUM(fp.outsourced_shipments)) OVER (PARTITION BY fp.half_year) AS mix_share,
    SUM(fp.vendor_cost) * 1.0 / SUM(fp.outsourced_shipments)                AS cost_per_shipment
FROM fact_vendor_performance fp
JOIN dim_vendor v ON fp.vendor_id = v.vendor_id
GROUP BY v.vendor_tier, fp.half_year;
-- CTE lớp 1: tính Cost/Shipment; lớp 2: pivot tính growth%
WITH cps AS (
    SELECT
        p.half_year,
        SUM(c.labor_cost)*1.0/SUM(p.shipment_volume)         AS labor_per_ship,
        SUM(c.transportation_cost)*1.0/SUM(p.shipment_volume) AS transport_per_ship,
        SUM(c.hub_handling_cost)*1.0/SUM(p.shipment_volume)   AS hub_per_ship
    FROM fact_delivery_performance p
    JOIN fact_delivery_cost c
        ON p.date=c.date AND p.hub_id=c.hub_id AND p.service_type=c.service_type
       AND p.route_type=c.route_type AND p.payment_type=c.payment_type AND p.size_type=c.size_type
    GROUP BY p.half_year
)
SELECT
    (MAX(CASE WHEN half_year='H1_2025' THEN labor_per_ship END)
     / MAX(CASE WHEN half_year='H2_2024' THEN labor_per_ship END)) - 1 AS labor_growth,
    (MAX(CASE WHEN half_year='H1_2025' THEN transport_per_ship END)
     / MAX(CASE WHEN half_year='H2_2024' THEN transport_per_ship END)) - 1 AS transport_growth,
    (MAX(CASE WHEN half_year='H1_2025' THEN hub_per_ship END)
     / MAX(CASE WHEN half_year='H2_2024' THEN hub_per_ship END)) - 1 AS hub_growth
FROM cps;
-- 5.1 Failed/Redelivery/RTS rate + Cost per Redelivery
SELECT
    p.half_year,
    SUM(p.first_attempt_failed_shipments)*1.0/SUM(p.shipment_volume) AS failed_rate,
    SUM(p.redelivery_shipments)*1.0/SUM(p.shipment_volume)           AS redelivery_rate,
    SUM(p.return_to_sender_shipments)*1.0/SUM(p.shipment_volume)     AS rts_rate,
    SUM(c.failed_redelivery_cost)*1.0/NULLIF(SUM(p.redelivery_shipments),0) AS cost_per_redelivery
FROM fact_delivery_performance p
JOIN fact_delivery_cost c ON p.date=c.date AND p.hub_id=c.hub_id
    AND p.service_type=c.service_type AND p.route_type=c.route_type
    AND p.payment_type=c.payment_type AND p.size_type=c.size_type
GROUP BY p.half_year;
-- 5.2 Breakdown theo payment_type và route_type
SELECT payment_type, half_year,
       SUM(first_attempt_failed_shipments)*1.0/SUM(shipment_volume) AS failed_rate
FROM fact_delivery_performance GROUP BY payment_type, half_year;

SELECT route_type, half_year,
       SUM(first_attempt_failed_shipments)*1.0/SUM(shipment_volume) AS failed_rate
FROM fact_delivery_performance GROUP BY route_type, half_year;
-- Cost/Shipment và Outsourced Share theo tháng (dùng DATE_TRUNC hoặc LEFT(date,7) tuỳ engine)
SELECT
    DATE_TRUNC('month', p.date) AS month,
    SUM(c.total_delivery_cost)*1.0/SUM(p.shipment_volume) AS cost_per_shipment,
    SUM(p.outsourced_shipments)*1.0/SUM(p.shipment_volume) AS outsourced_share
FROM fact_delivery_performance p
JOIN fact_delivery_cost c ON p.date=c.date AND p.hub_id=c.hub_id
    AND p.service_type=c.service_type AND p.route_type=c.route_type
    AND p.payment_type=c.payment_type AND p.size_type=c.size_type
GROUP BY DATE_TRUNC('month', p.date)
ORDER BY month;
-- 7.1 Misroute Rate theo Focus/Normal
SELECT h.is_focus_hub, r.half_year, AVG(r.misroute_rate) AS misroute_rate
FROM fact_rider_hub_productivity r
JOIN dim_hub h ON r.hub_id = h.hub_id
GROUP BY h.is_focus_hub, r.half_year;
-- 7.2 Productivity (Shipments/Rider) theo nhóm
SELECT h.is_focus_hub, r.half_year, AVG(r.shipments_per_rider) AS avg_shipments_per_rider
FROM fact_rider_hub_productivity r
JOIN dim_hub h ON r.hub_id = h.hub_id
GROUP BY h.is_focus_hub, r.half_year;
-- 7.3 Size Type mix share + cost/shipment — loại trừ A2
SELECT
    size_type, half_year,
    SUM(shipment_volume) * 1.0 / SUM(SUM(shipment_volume)) OVER (PARTITION BY half_year) AS mix_share
FROM fact_delivery_performance
GROUP BY size_type, half_year;
-- 7.4 Outsourced Share theo Payment x Route — loại trừ né COD
SELECT payment_type, route_type, half_year,
       SUM(outsourced_shipments)*1.0/SUM(shipment_volume) AS outsourced_share
FROM fact_delivery_performance
GROUP BY payment_type, route_type, half_year;
-- 7.5 Region & Open Date
SELECT h.region, p.half_year,
       SUM(p.outsourced_shipments)*1.0/SUM(p.shipment_volume) AS outsourced_share
FROM fact_delivery_performance p JOIN dim_hub h ON p.hub_id=h.hub_id
GROUP BY h.region, p.half_year;

SELECT h.hub_name, h.open_date, h.is_focus_hub FROM dim_hub h ORDER BY h.open_date;
-- C2: COD rejection — RTS & Failed rate theo payment_type
SELECT payment_type, half_year,
       SUM(return_to_sender_shipments)*1.0/SUM(shipment_volume) AS rts_rate,
       SUM(first_attempt_failed_shipments)*1.0/SUM(shipment_volume) AS failed_rate
FROM fact_delivery_performance
GROUP BY payment_type, half_year;
-- C4: Delivery promise mismatch — Late rate + SLA target vs Actual theo service_type
SELECT service_type, half_year,
       SUM(late_deliveries)*1.0/SUM(shipment_volume)                         AS late_rate,
       SUM(sla_target_days*shipment_volume)*1.0/SUM(shipment_volume)         AS avg_sla_target,
       SUM(avg_delivery_days*shipment_volume)*1.0/SUM(shipment_volume)       AS avg_actual_days
FROM fact_delivery_performance
GROUP BY service_type, half_year;
-- Hub Staff Count vs Rider Count growth, theo Focus/Normal — phát hiện "tăng rider, cắt staff kho"
SELECT
    h.is_focus_hub,
    (SUM(CASE WHEN r.half_year='H1_2025' THEN r.rider_count END)*1.0
     / SUM(CASE WHEN r.half_year='H2_2024' THEN r.rider_count END)) - 1      AS rider_growth,
    (SUM(CASE WHEN r.half_year='H1_2025' THEN r.hub_staff_count END)*1.0
     / SUM(CASE WHEN r.half_year='H2_2024' THEN r.hub_staff_count END)) - 1  AS hub_staff_growth
FROM fact_rider_hub_productivity r
JOIN dim_hub h ON r.hub_id = h.hub_id
GROUP BY h.is_focus_hub;
-- Cost/Shipment growth theo component, tách Focus vs Normal (test B1/B3/B4 đồng thời)
SELECT
    h.is_focus_hub,
    (SUM(CASE WHEN c.half_year='H1_2025' THEN c.labor_cost END)*1.0
     / SUM(CASE WHEN c.half_year='H1_2025' THEN p.shipment_volume END))
    / (SUM(CASE WHEN c.half_year='H2_2024' THEN c.labor_cost END)*1.0
     / SUM(CASE WHEN c.half_year='H2_2024' THEN p.shipment_volume END)) - 1 AS labor_per_ship_growth
FROM fact_delivery_cost c
JOIN fact_delivery_performance p ON c.date=p.date AND c.hub_id=p.hub_id
    AND c.service_type=p.service_type AND c.route_type=p.route_type
    AND c.payment_type=p.payment_type AND c.size_type=p.size_type
JOIN dim_hub h ON c.hub_id = h.hub_id
GROUP BY h.is_focus_hub;
-- (lặp lại tương tự cho transportation_cost, hub_handling_cost)
-- Cost per On-time Delivery — con số chốt hạ trả lời business question gốc
SELECT
    p.half_year,
    SUM(c.total_delivery_cost) * 1.0 / SUM(p.on_time_deliveries) AS cost_per_ontime_delivery
FROM fact_delivery_performance p
JOIN fact_delivery_cost c ON p.date=c.date AND p.hub_id=c.hub_id
    AND p.service_type=c.service_type AND p.route_type=c.route_type
    AND p.payment_type=c.payment_type AND p.size_type=c.size_type
GROUP BY p.half_year;
