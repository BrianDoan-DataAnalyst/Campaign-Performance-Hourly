# Campaign-Performance-Hourly
Top 100 SKU by hour
![image](https://user-images.githubusercontent.com/89034147/135488309-3804fee5-ab7c-40a1-9da2-f46bca34f10d.png)


Soucre code:
CREATE view v_campaign_performance_hour as (


with  t2 as (	
					
 -- t2 dùng để combine sku_id con với sku_id lớn chung 1 cột SHOPEE					
	with s1 as (
	
		 SELECT 
					date(tss.date) as date,
					extract( hour from date) as hour,
					tss.shop_account,
					tss.sku,
					product_name,
					product_id,
					placed_units as net_order,
					unit_sold,
					placed_sales as revenue,
					sku_views as product_page_view,
					add_to_cart_unit as add_to_cart_units, 
					add_to_cart_visitor,
					uv_to_placed_buyers_rate as conversion_rate_page_view, --Tỷ lệ chuyển đổi (Lượt truy cập - Đơn được đặt)
					uv_to_paid_buyers_rate as conversion_rate_visitor, --Tỷ lệ chuyển đổi (Đơn đã thanh toán /Lượt truy cập)
					'SHOPEE' as platform,
					count(product_id) OVER
					(PARTITION BY product_id,date) as count_product_id,
					concat(product_id,'-',sku) as sku_type
					
		FROM athena_warehouse.traffic_sku_shopee_metrics tss
		WHERE date(tss.date) >= CURRENT_DATE - INTERVAL '2 months' -- Lấy dữ liệu 1 tháng gần nhất
					AND date(tss.date)>='2021-09-10' ), --SUCCESS S1
					
					s2 as (SELECT *,
						CASE 
									WHEN sku='nan'  AND count_product_id=1
									THEN product_id 
									WHEN sku='-' AND count_product_id=1
									THEN product_id
									WHEN sku='-' AND count_product_id>1
									THEN '-'
									ELSE sku_type
									END AS sku_id
						FROM s1 ) ,
						
					s3 as (SELECT date,product_id,sku, product_page_view FROM s1 WHERE sku='nan' or sku='-') -- Loại các sku_id nhỏ không có số page_view
					
		       SELECT s2.date,
									s2.hour,
									shop_account,
									t1.sku as sku_id,
									product_name,
									s2.product_id,
									net_order,
									unit_sold,
									revenue,
									s3.product_page_view,
									add_to_cart_units, 
									add_to_cart_visitor,
									conversion_rate_page_view, --Tỷ lệ chuyển đổi (Lượt truy cập - Đơn được đặt)
									conversion_rate_visitor, --Tỷ lệ chuyển đổi (Đơn đã thanh toán /Lượt truy cập)
									s2.platform
						FROM s2 
						LEFT JOIN s3 ON s2.date=s3.date AND s2.product_id=s3.product_id
						LEFT JOIN athena_public.main_opollo_products t1
							ON s2.sku_id=t1.platform_sku	
													AND upper(t1.platform) ='SHOPEE'
													AND upper(t1.sku) NOT LIKE '%OLD%' 
													AND upper(t1.sku) NOT LIKE '%NEW%' 
													AND upper(t1.sku) NOT LIKE '%A' 
													AND (upper(t1.sku) NOT LIKE '%!_%' ESCAPE '!')  ), -- SUCCESS T2
									
	t3 as		
	(
		(SELECT 
		  date(monitor_time) as date,
			extract(hour from monitor_time ) as hour,
			btp.shop_account,
			t1.sku as sku_id,
			product_name,
			product_id,
			total_orders as net_order,
			sold_units as unit_sold,
			revenues as revenue,
			total_view_product as product_page_view,
			total_add_to_cart_product as add_to_cart_units,
			0 as add_to_cart_visitor,
			conversion_rate as conversion_rate_page_view,
			--0 as conversion_rate_buyer,
			'TIKI' as platform
		FROM athena_public.ba_tiki_product_realtime btp
		LEFT JOIN athena_public.main_opollo_products t1
			ON btp.sku= t1.platform_sku 
					AND t1.Platform ='TIKI'
					AND upper(t1.sku) NOT LIKE '%OLD%' 
					AND upper(t1.sku) NOT LIKE '%NEW%' 
					AND upper(t1.sku) NOT LIKE '%A' 
					AND (upper(t1.sku) NOT LIKE '%!_%' ESCAPE '!')
		WHERE date(monitor_time) >= CURRENT_DATE - INTERVAL '1 months' -- Lấy 2 tháng gần nhất
					AND date(monitor_time)>='2021-09-10') -- Dữ liệu update đầy đủ từ '2021-09-10'
		UNION ALL
		(SELECT 
			date(tsl.date) as date,
			extract(hour from tsl.date) as hour,
			tsl.shop_account,
			seller_sku as sku_id,
			product_name,
			product_id,
			orders as net_order,
			unit_sold,
			revenue,
			sku_views as product_page_view,
			add_to_cart_unit as add_to_cart_units,
			add_to_cart_visitor, 
			round((orders/NULLIF(sku_views,0)),4) as conversion_rate_page_view, --Tỷ lệ chuyển đổi (Lượt truy cập - Đơn được đặt)
			--buyer_conversion_rate as conversion_rate_buyer,
			'LZD' as platform
		FROM athena_warehouse.traffic_sku_lazada_metrics_realtime tsl
		WHERE date(tsl.date) >= CURRENT_DATE - INTERVAL '1 months' --Lấy data 1 tháng gần nhất
					AND date(tsl.date)>='2021-09-10' -- Dữ liệu update đầy đủ từ '2021-09-10' ) -- LZD
					
		UNION ALL
		
		(SELECT t2.date,
						t2.hour,
						shop_account,
						sku_id,
						product_name,
						product_id,
						net_order,
						unit_sold,
						revenue,
						product_page_view,
						add_to_cart_units, 
						add_to_cart_visitor,
						conversion_rate_page_view, --Tỷ lệ chuyển đổi (Lượt truy cập - Đơn được đặt)
						--conversion_rate_visitor, --Tỷ lệ chuyển đổi (Đơn đã thanh toán /Lượt truy cập)
						platform
			FROM t2 )  ) ), -- SUCCESS t3

-- t4 dùng để lấy NMV và GMV
	t4 as 
	( with m1 as (
			  SELECT date(order_created_at) as date,
						extract(hour from order_created_at) as hour,
						platform,
						sku,
						brand,
						product_id,
						sum(v_nmv_calculation(payment_type , platform , order_created_at , status , order_type , selling_price , paid_price , sku , sku_name,platform_voucher , quantity , income , platform_fee , subsidy , seller_voucher , order_value,original_price)) as nmv_usd,
						sum(selling_price*quantity) as gmv
		   FROM athena_public.main_order2019 
		   WHERE status NOT IN ('Canceled','Returned','Lost','Failed', 'Returning')
				    AND brand <> 'MOIRA'
				    AND order_created_at >= CURRENT_DATE - INTERVAL '1 months' AND DATE(order_created_at)>= '2021-09-10'
				    AND platform IN ('TIKI', 'LZD','SHOPEE')
		   GROUP BY date(order_created_at),extract(hour from order_created_at),platform,sku,brand,product_id  ),--SUCCESS M1
			 
		--m2 dùng để xếp hạng sku theo nmv cao nhất trong ngày	
			m2 as	(
						 SELECT *,
										RANK () OVER 
										(PARTITION BY HOUR, DATE ORDER BY nmv_usd DESC) as rank_nmv
							FROM m1 ) -- SUCCESS M2
							
			SELECT * FROM m2 WHERE rank_nmv <=100  ) -- SUCCESS T4
	
	SELECT t3.date,
							 t3.hour,
							 t3.sku_id,
							 product_name,
							 t3.product_id,
							 net_order,
							 unit_sold,
							 revenue,
							 product_page_view,
							 --product_unique_page_view,
							 add_to_cart_units,
							 add_to_cart_visitor,
							 conversion_rate_page_view,
							 --conversion_rate_visitor,
							 --conversion_rate_buyer,
							 t3.shop_account,
							 v.brand,
							 v.groupbrand,
							 t3.platform,
							 t4.nmv_usd,
							 t4.gmv,
							 t4.rank_nmv
				FROM t3 
				LEFT JOIN t4 ON t3.date=t4.date AND t3.hour=t4.hour AND t3.platform=t4.platform AND t3.sku_id=t4.sku 
				LEFT JOIN athena_public.v_channel_code_and_groupbrand_list as v 
				ON v.shop_account = t3.shop_account AND v.platform=t3.platform
				WHERE t4.rank_nmv>0 AND t4.rank_nmv<=100 ); -- Lấy top 100 sku có nmv cao nhất theo từng giờ của mỗi ngày ------ END
