# Answering Business Questions Using SQL

We are going to answer business questions using SQL skills.


```python
%%capture
%load_ext sql
%sql sqlite:///chinook.db
```




    'Connected: None@chinook.db'



### Overview of the Data


```sql
%%sql
SELECT name,type
FROM sqlite_master
WHERE type IN ("table","view")
```

    Done.
    




<table>
    <tr>
        <th>name</th>
        <th>type</th>
    </tr>
    <tr>
        <td>album</td>
        <td>table</td>
    </tr>
    <tr>
        <td>artist</td>
        <td>table</td>
    </tr>
    <tr>
        <td>customer</td>
        <td>table</td>
    </tr>
    <tr>
        <td>employee</td>
        <td>table</td>
    </tr>
    <tr>
        <td>genre</td>
        <td>table</td>
    </tr>
    <tr>
        <td>invoice</td>
        <td>table</td>
    </tr>
    <tr>
        <td>invoice_line</td>
        <td>table</td>
    </tr>
    <tr>
        <td>media_type</td>
        <td>table</td>
    </tr>
    <tr>
        <td>playlist</td>
        <td>table</td>
    </tr>
    <tr>
        <td>playlist_track</td>
        <td>table</td>
    </tr>
    <tr>
        <td>track</td>
        <td>table</td>
    </tr>
</table>



## Schema Diagram

<img src='chinook-schema.svg'>

## Which genre sells the best in the USA?


```sql
%%sql
WITH usa_invoices AS(
    SELECT 
        il.*
    FROM invoice_line il
    LEFT JOIN invoice i ON i.invoice_id = il.invoice_id
    LEFT JOIN customer c ON c.customer_id = i.customer_id
    WHERE c.country LIKE '%usa%'
),genre_cte AS(
    SELECT t.track_id,g.name
    FROM track t
    LEFT JOIN genre g ON t.genre_id = g.genre_id
)
SELECT
    genre_cte.name genre,
    COUNT(usa_invoices.invoice_id) track_sold,
    ROUND(COUNT(usa_invoices.invoice_id) *100.0/(
        SELECT COUNT(*)
        FROM usa_invoices
    ),2) sold_percentage
FROM usa_invoices
LEFT JOIN genre_cte ON usa_invoices.track_id = genre_cte.track_id
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10
```

    Done.
    




<table>
    <tr>
        <th>genre</th>
        <th>track_sold</th>
        <th>sold_percentage</th>
    </tr>
    <tr>
        <td>Rock</td>
        <td>561</td>
        <td>53.38</td>
    </tr>
    <tr>
        <td>Alternative &amp; Punk</td>
        <td>130</td>
        <td>12.37</td>
    </tr>
    <tr>
        <td>Metal</td>
        <td>124</td>
        <td>11.8</td>
    </tr>
    <tr>
        <td>R&amp;B/Soul</td>
        <td>53</td>
        <td>5.04</td>
    </tr>
    <tr>
        <td>Blues</td>
        <td>36</td>
        <td>3.43</td>
    </tr>
    <tr>
        <td>Alternative</td>
        <td>35</td>
        <td>3.33</td>
    </tr>
    <tr>
        <td>Latin</td>
        <td>22</td>
        <td>2.09</td>
    </tr>
    <tr>
        <td>Pop</td>
        <td>22</td>
        <td>2.09</td>
    </tr>
    <tr>
        <td>Hip Hop/Rap</td>
        <td>20</td>
        <td>1.9</td>
    </tr>
    <tr>
        <td>Jazz</td>
        <td>14</td>
        <td>1.33</td>
    </tr>
</table>



Note that Rock genre accounts for 53% of sales we should lookout for artist or albums of Rock genre.

## Analyzing Employee Sales Performance

Each customer for the Chinook store get assigned to a sales support agent within the company when they first make purchase.


```sql
%%sql
WITH sales_cte AS(
SELECT
    customer_id,
    SUM(total) total,
    (
        SELECT 
         support_rep_id
        FROM customer
        WHERE invoice.customer_id = customer.customer_id
    ) support_rep_id
FROM invoice
GROUP BY 1
)

SELECT 
    e.employee_id,
    e.first_name || ' ' || e.last_name full_name,
    SUM(s.total) sales_total,
    hire_date
FROM employee e
LEFT JOIN sales_cte s ON e.employee_id = s.support_rep_id 
GROUP BY 1
ORDER BY 3 DESC


```

    Done.
    




<table>
    <tr>
        <th>employee_id</th>
        <th>full_name</th>
        <th>sales_total</th>
        <th>hire_date</th>
    </tr>
    <tr>
        <td>3</td>
        <td>Jane Peacock</td>
        <td>1731.51</td>
        <td>2017-04-01 00:00:00</td>
    </tr>
    <tr>
        <td>4</td>
        <td>Margaret Park</td>
        <td>1584.0</td>
        <td>2017-05-03 00:00:00</td>
    </tr>
    <tr>
        <td>5</td>
        <td>Steve Johnson</td>
        <td>1393.92</td>
        <td>2017-10-17 00:00:00</td>
    </tr>
    <tr>
        <td>1</td>
        <td>Andrew Adams</td>
        <td>None</td>
        <td>2016-08-14 00:00:00</td>
    </tr>
    <tr>
        <td>2</td>
        <td>Nancy Edwards</td>
        <td>None</td>
        <td>2016-05-01 00:00:00</td>
    </tr>
    <tr>
        <td>6</td>
        <td>Michael Mitchell</td>
        <td>None</td>
        <td>2016-10-17 00:00:00</td>
    </tr>
    <tr>
        <td>7</td>
        <td>Robert King</td>
        <td>None</td>
        <td>2017-01-02 00:00:00</td>
    </tr>
    <tr>
        <td>8</td>
        <td>Laura Callahan</td>
        <td>None</td>
        <td>2017-03-04 00:00:00</td>
    </tr>
</table>



Only 3 employee seems active and is reflected on their sales performance.

## Analyzing Sales by Country

Next task is to analyze the sales data from customers for each different country. We use the country value from `customers` table.


```sql
%%sql
CREATE VIEW country_sales_table AS 
SELECT 
    c.country,    
    s.*
FROM (
     SELECT
        customer_id,
        SUM(total) sales_sum,
        COUNT(invoice_id) order_count
    FROM invoice
    GROUP BY 1
) s
LEFT JOIN customer c ON s.customer_id = c.customer_id

```

    Done.
    




    []




```sql
%%sql
SELECT country,
SUM(sales_sum) total_sales,
COUNT(DISTINCT customer_id) unique_customer
FROM country_sales_table
GROUP BY 1
ORDER BY 2 DESC
```

    Done.
    




<table>
    <tr>
        <th>country</th>
        <th>total_sales</th>
        <th>unique_customer</th>
    </tr>
    <tr>
        <td>USA</td>
        <td>1040.49</td>
        <td>13</td>
    </tr>
    <tr>
        <td>Canada</td>
        <td>535.59</td>
        <td>8</td>
    </tr>
    <tr>
        <td>Brazil</td>
        <td>427.67999999999995</td>
        <td>5</td>
    </tr>
    <tr>
        <td>France</td>
        <td>389.07</td>
        <td>5</td>
    </tr>
    <tr>
        <td>Germany</td>
        <td>334.62</td>
        <td>4</td>
    </tr>
    <tr>
        <td>Czech Republic</td>
        <td>273.24</td>
        <td>2</td>
    </tr>
    <tr>
        <td>United Kingdom</td>
        <td>245.51999999999998</td>
        <td>3</td>
    </tr>
    <tr>
        <td>Portugal</td>
        <td>185.13</td>
        <td>2</td>
    </tr>
    <tr>
        <td>India</td>
        <td>183.14999999999998</td>
        <td>2</td>
    </tr>
    <tr>
        <td>Ireland</td>
        <td>114.83999999999997</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Spain</td>
        <td>98.01</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Chile</td>
        <td>97.02000000000001</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Australia</td>
        <td>81.18</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Finland</td>
        <td>79.2</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Hungary</td>
        <td>78.21</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Poland</td>
        <td>76.22999999999999</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Sweden</td>
        <td>75.24</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Norway</td>
        <td>72.27000000000001</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Austria</td>
        <td>69.3</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Netherlands</td>
        <td>65.34</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Belgium</td>
        <td>60.38999999999999</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Italy</td>
        <td>50.49</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Argentina</td>
        <td>39.6</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Denmark</td>
        <td>37.61999999999999</td>
        <td>1</td>
    </tr>
</table>



Some country has only one customer. We will group them into 'Other' group


```sql
%%sql

WITH country_with_other AS(
    SELECT
        CASE WHEN (SELECT COUNT(DISTINCT customer_id) 
                   FROM country_sales_table t2
                  WHERE t1.country = t2.country) = 1 
        THEN 'Other' ELSE t1.country END AS Country,
        t1.customer_id,
        t1.sales_sum,
        t1.order_count
    FROM country_sales_table t1
)
SELECT
    s1.*,
    CASE WHEN country = 'Other' 
    THEN 0 ELSE 1 END AS rank
FROM(
    SELECT 
        country,
        COUNT(DISTINCT customer_id) num_customer,
        SUM(sales_sum) total_sum,
        ROUND(AVG(sales_sum),2) avg_sales,
        ROUND(SUM(sales_sum)/SUM(order_count),2) avg_order_value
    FROM country_with_other
    GROUP BY country
    ORDER BY 3 DESC
) s1
ORDER BY rank DESC
```

    Done.
    




<table>
    <tr>
        <th>country</th>
        <th>num_customer</th>
        <th>total_sum</th>
        <th>avg_sales</th>
        <th>avg_order_value</th>
        <th>rank</th>
    </tr>
    <tr>
        <td>USA</td>
        <td>13</td>
        <td>1040.49</td>
        <td>80.04</td>
        <td>7.94</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Canada</td>
        <td>8</td>
        <td>535.59</td>
        <td>66.95</td>
        <td>7.05</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Brazil</td>
        <td>5</td>
        <td>427.67999999999995</td>
        <td>85.54</td>
        <td>7.01</td>
        <td>1</td>
    </tr>
    <tr>
        <td>France</td>
        <td>5</td>
        <td>389.07</td>
        <td>77.81</td>
        <td>7.78</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Germany</td>
        <td>4</td>
        <td>334.62</td>
        <td>83.66</td>
        <td>8.16</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Czech Republic</td>
        <td>2</td>
        <td>273.24</td>
        <td>136.62</td>
        <td>9.11</td>
        <td>1</td>
    </tr>
    <tr>
        <td>United Kingdom</td>
        <td>3</td>
        <td>245.51999999999998</td>
        <td>81.84</td>
        <td>8.77</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Portugal</td>
        <td>2</td>
        <td>185.13</td>
        <td>92.56</td>
        <td>6.38</td>
        <td>1</td>
    </tr>
    <tr>
        <td>India</td>
        <td>2</td>
        <td>183.14999999999998</td>
        <td>91.57</td>
        <td>8.72</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Other</td>
        <td>15</td>
        <td>1094.94</td>
        <td>73.0</td>
        <td>7.45</td>
        <td>0</td>
    </tr>
</table>



## Albums Vs Individual Tracks

Chinook store allows customer to make purchases in one in two ways
1. purchase whole album
2. purchase a collection of one or more individual tracks

We want to find what % of purchases are individual tracks vs whole albms. so Chinook Management can use this data in considering to purchase only the most popular tracks from each album instead of purchasing every track from an album and its effect on overall revenue.


```sql
%%sql
WITH invoice_first_track AS(SELECT
    il.invoice_id ,
    MIN(track_id) first_track_id
FROM invoice_line il
GROUP BY 1),
album_purchase_cte AS(
SELECT
    ifs.*,
    CASE WHEN(
        SELECT 
            t1.track_id
        FROM track t1
        WHERE t1.album_id = (
            SELECT album_id
            FROM track t2
            WHERE ifs.first_track_id = t2.track_id
        )
        EXCEPT
        
        SELECT
            track_id
        FROM invoice_line il2
        WHERE il2.invoice_id = ifs.invoice_id
    ) IS NULL
    AND (
        SELECT
            track_id
        FROM invoice_line il2
        WHERE il2.invoice_id = ifs.invoice_id
        EXCEPT
        SELECT 
            t1.track_id
        FROM track t1
        WHERE t1.album_id = (
            SELECT album_id
            FROM track t2
            WHERE ifs.first_track_id = t2.track_id
        )
    ) IS NULL 
    THEN 'yes'
    ELSE 'no'
    END AS album_purchase
FROM invoice_first_track ifs
)
SELECT album_purchase "Album purchase ?",
    COUNT(invoice_id) "order_count",
    ROUND(COUNT(invoice_id)*100.0/(SELECT COUNT(*) FROM invoice),2) AS "% of purchases"
FROM album_purchase_cte
GROUP BY 1
```

    Done.
    




<table>
    <tr>
        <th>Album purchase ?</th>
        <th>order_count</th>
        <th>% of purchases</th>
    </tr>
    <tr>
        <td>no</td>
        <td>500</td>
        <td>81.43</td>
    </tr>
    <tr>
        <td>yes</td>
        <td>114</td>
        <td>18.57</td>
    </tr>
</table>



Majority 82% purchase is single track. While whole album purchase account for 18% only. Based on this finding, Chinook should consider purchasing popular tracks of albums from record companies.

## Which artist is used in the most playlist ?


```sql
%%sql
SELECT count(*) records,
COUNT(DISTINCT playlist_id) 'Unique playlist', 
COUNT(DISTINCT track_id) 'Unique tracks'
FROM playlist_track

```

    Done.
    




<table>
    <tr>
        <th>records</th>
        <th>Unique playlist</th>
        <th>Unique tracks</th>
    </tr>
    <tr>
        <td>8715</td>
        <td>14</td>
        <td>3503</td>
    </tr>
</table>




```sql
%%sql
WITH artist_name AS(
SELECT 
    t.track_id track_id,
    ar.name artist_name,
    g.name genre_name
FROM track t
LEFT JOIN album al ON al.album_id = t.album_id
LEFT JOIN artist ar ON ar.artist_id = al.artist_id
LEFT JOIN genre g ON g.genre_id = t.genre_id
)

SELECT
    artist_name,
    COUNT(playlist_id) times_appeared,
    ROUND(COUNT(playlist_id)*100.0/(
        SELECT COUNT(*)
        FROM playlist_track
    ),3) perc_appeared,
    genre_name 
FROM(
    SELECT 
        pt.playlist_id,
        pt.track_id,
        an.artist_name,
        an.genre_name
    FROM playlist_track pt
    LEFT JOIN artist_name an ON an.track_id = pt.track_id
    ) playlist_artist
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10
```

    Done.
    




<table>
    <tr>
        <th>artist_name</th>
        <th>times_appeared</th>
        <th>perc_appeared</th>
        <th>genre_name</th>
    </tr>
    <tr>
        <td>Iron Maiden</td>
        <td>516</td>
        <td>5.921</td>
        <td>Metal</td>
    </tr>
    <tr>
        <td>U2</td>
        <td>333</td>
        <td>3.821</td>
        <td>Pop</td>
    </tr>
    <tr>
        <td>Metallica</td>
        <td>296</td>
        <td>3.396</td>
        <td>Metal</td>
    </tr>
    <tr>
        <td>Led Zeppelin</td>
        <td>252</td>
        <td>2.892</td>
        <td>Rock</td>
    </tr>
    <tr>
        <td>Deep Purple</td>
        <td>226</td>
        <td>2.593</td>
        <td>Rock</td>
    </tr>
    <tr>
        <td>Lost</td>
        <td>184</td>
        <td>2.111</td>
        <td>Drama</td>
    </tr>
    <tr>
        <td>Pearl Jam</td>
        <td>177</td>
        <td>2.031</td>
        <td>Rock</td>
    </tr>
    <tr>
        <td>Eric Clapton</td>
        <td>145</td>
        <td>1.664</td>
        <td>Latin</td>
    </tr>
    <tr>
        <td>Faith No More</td>
        <td>145</td>
        <td>1.664</td>
        <td>Alternative &amp; Punk</td>
    </tr>
    <tr>
        <td>Lenny Kravitz</td>
        <td>143</td>
        <td>1.641</td>
        <td>Metal</td>
    </tr>
</table>



## How many Tracks have been purchased vs Not purchased ? 


```sql
%%sql
WITH track_purchased AS(
    SELECT
        DISTINCT track_id
    FROM invoice_line
)
SELECT
    purchased_or_not 'Track purchased ? ',
    COUNT(track_id) num_track,
    ROUND(COUNT(track_id)*100.0/(
        SELECT COUNT(*)
        FROM track
    ),2) perc_track
FROM(
    SELECT 
        track_id,
        CASE WHEN track_id IN (
            SELECT *
            FROM track_purchased
        ) THEN 'Yes'
        ELSE 'No'
        END AS purchased_or_not
    FROM track
)
GROUP BY 1
ORDER BY 2 DESC
```

    Done.
    




<table>
    <tr>
        <th>Track purchased ? </th>
        <th>num_track</th>
        <th>perc_track</th>
    </tr>
    <tr>
        <td>Yes</td>
        <td>1806</td>
        <td>51.56</td>
    </tr>
    <tr>
        <td>No</td>
        <td>1697</td>
        <td>48.44</td>
    </tr>
</table>



## Do pretected vs. non protected media types have an effect on popularity?


```sql
%%sql
WITH media_cte AS(
    SELECT
        mt.name AS media,
        ROUND(SUM(i.total),2) sold_amount,
        COUNT(i.invoice_id) tracks_sold,
        COUNT(*) total_tracks
    FROM media_type mt
    LEFT JOIN track t
    ON mt.media_type_id = t.media_type_id
    LEFT JOIN invoice_line il
    ON t.track_id = il.track_id  
    LEFT JOIN invoice i
    ON i.invoice_id = il.invoice_id
    GROUP BY 1
), media_count_cte AS(
    SELECT
     media_type_id media_id,   
     count(*) total_tracks
    FROM track
    GROUP BY 1
)

SELECT *,
    ROUND(tracks_sold *100.0/total_tracks,2) perc_sold
FROM media_cte
ORDER BY tracks_sold DESC

```

    Done.
    




<table>
    <tr>
        <th>media</th>
        <th>sold_amount</th>
        <th>tracks_sold</th>
        <th>total_tracks</th>
        <th>perc_sold</th>
    </tr>
    <tr>
        <td>MPEG audio file</td>
        <td>42934.32</td>
        <td>4259</td>
        <td>5652</td>
        <td>75.35</td>
    </tr>
    <tr>
        <td>Protected AAC audio file</td>
        <td>4115.43</td>
        <td>439</td>
        <td>525</td>
        <td>83.62</td>
    </tr>
    <tr>
        <td>Purchased AAC audio file</td>
        <td>274.23</td>
        <td>35</td>
        <td>39</td>
        <td>89.74</td>
    </tr>
    <tr>
        <td>AAC audio file</td>
        <td>153.45</td>
        <td>21</td>
        <td>24</td>
        <td>87.5</td>
    </tr>
    <tr>
        <td>Protected MPEG-4 video file</td>
        <td>25.74</td>
        <td>3</td>
        <td>214</td>
        <td>1.4</td>
    </tr>
</table>



## Is the range of tracks in the store reflective of their sales popularity?


```sql
%%sql

SELECT *,
    ROUND(tracks_sold*100.0/total_tracks,2) perc_sold,
    total_tracks - tracks_sold dif
FROM (
    SELECT 
        g.name,
        COUNT(t.track_id) total_tracks,
        COUNT(il.track_id) tracks_sold,
        ROUND(SUM(total),2) sold_amount
    FROM track t
    LEFT JOIN genre g
    ON t.genre_id = g.genre_id
    LEFT JOIN invoice_line il
    ON t.track_id = il.track_id
    LEFT JOIN invoice i
    ON i.invoice_id = il.invoice_id
    GROUP BY 1
)
ORDER BY perc_sold DESC
```

    Done.
    




<table>
    <tr>
        <th>name</th>
        <th>total_tracks</th>
        <th>tracks_sold</th>
        <th>sold_amount</th>
        <th>perc_sold</th>
        <th>dif</th>
    </tr>
    <tr>
        <td>Easy Listening</td>
        <td>74</td>
        <td>74</td>
        <td>951.39</td>
        <td>100.0</td>
        <td>0</td>
    </tr>
    <tr>
        <td>Electronica/Dance</td>
        <td>56</td>
        <td>55</td>
        <td>614.79</td>
        <td>98.21</td>
        <td>1</td>
    </tr>
    <tr>
        <td>R&amp;B/Soul</td>
        <td>165</td>
        <td>159</td>
        <td>1751.31</td>
        <td>96.36</td>
        <td>6</td>
    </tr>
    <tr>
        <td>Alternative</td>
        <td>123</td>
        <td>117</td>
        <td>1095.93</td>
        <td>95.12</td>
        <td>6</td>
    </tr>
    <tr>
        <td>Rock</td>
        <td>3017</td>
        <td>2635</td>
        <td>26751.78</td>
        <td>87.34</td>
        <td>382</td>
    </tr>
    <tr>
        <td>Blues</td>
        <td>149</td>
        <td>124</td>
        <td>1379.07</td>
        <td>83.22</td>
        <td>25</td>
    </tr>
    <tr>
        <td>Metal</td>
        <td>755</td>
        <td>619</td>
        <td>5316.3</td>
        <td>81.99</td>
        <td>136</td>
    </tr>
    <tr>
        <td>Alternative &amp; Punk</td>
        <td>648</td>
        <td>492</td>
        <td>4841.1</td>
        <td>75.93</td>
        <td>156</td>
    </tr>
    <tr>
        <td>Pop</td>
        <td>86</td>
        <td>63</td>
        <td>568.26</td>
        <td>73.26</td>
        <td>23</td>
    </tr>
    <tr>
        <td>Hip Hop/Rap</td>
        <td>47</td>
        <td>33</td>
        <td>463.32</td>
        <td>70.21</td>
        <td>14</td>
    </tr>
    <tr>
        <td>Jazz</td>
        <td>190</td>
        <td>121</td>
        <td>1302.84</td>
        <td>63.68</td>
        <td>69</td>
    </tr>
    <tr>
        <td>Reggae</td>
        <td>71</td>
        <td>35</td>
        <td>257.4</td>
        <td>49.3</td>
        <td>36</td>
    </tr>
    <tr>
        <td>Classical</td>
        <td>105</td>
        <td>47</td>
        <td>361.35</td>
        <td>44.76</td>
        <td>58</td>
    </tr>
    <tr>
        <td>Heavy Metal</td>
        <td>29</td>
        <td>8</td>
        <td>70.29</td>
        <td>27.59</td>
        <td>21</td>
    </tr>
    <tr>
        <td>Latin</td>
        <td>627</td>
        <td>167</td>
        <td>1705.77</td>
        <td>26.63</td>
        <td>460</td>
    </tr>
    <tr>
        <td>Soundtrack</td>
        <td>43</td>
        <td>5</td>
        <td>46.53</td>
        <td>11.63</td>
        <td>38</td>
    </tr>
    <tr>
        <td>TV Shows</td>
        <td>93</td>
        <td>2</td>
        <td>19.8</td>
        <td>2.15</td>
        <td>91</td>
    </tr>
    <tr>
        <td>Drama</td>
        <td>64</td>
        <td>1</td>
        <td>5.94</td>
        <td>1.56</td>
        <td>63</td>
    </tr>
    <tr>
        <td>Bossa Nova</td>
        <td>15</td>
        <td>0</td>
        <td>None</td>
        <td>0.0</td>
        <td>15</td>
    </tr>
    <tr>
        <td>Comedy</td>
        <td>17</td>
        <td>0</td>
        <td>None</td>
        <td>0.0</td>
        <td>17</td>
    </tr>
    <tr>
        <td>Opera</td>
        <td>1</td>
        <td>0</td>
        <td>None</td>
        <td>0.0</td>
        <td>1</td>
    </tr>
    <tr>
        <td>Rock And Roll</td>
        <td>12</td>
        <td>0</td>
        <td>None</td>
        <td>0.0</td>
        <td>12</td>
    </tr>
    <tr>
        <td>Sci Fi &amp; Fantasy</td>
        <td>26</td>
        <td>0</td>
        <td>None</td>
        <td>0.0</td>
        <td>26</td>
    </tr>
    <tr>
        <td>Science Fiction</td>
        <td>13</td>
        <td>0</td>
        <td>None</td>
        <td>0.0</td>
        <td>13</td>
    </tr>
    <tr>
        <td>World</td>
        <td>28</td>
        <td>0</td>
        <td>None</td>
        <td>0.0</td>
        <td>28</td>
    </tr>
</table>



Rock genre is the popular among customer. There are more tracks in this genre and the sales is reflected in sales. There are some genre preferred by customer though the range of tracks is limited but score higher sales %.
