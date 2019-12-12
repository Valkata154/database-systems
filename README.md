# Section One
## Answer the following questions:

### 1. Trip 73440. How many items were transported during trip 73440?
``` sql
+-------+
| Items |
+-------+
| 19    |
+-------+

SELECT COUNT(barcode) AS Items 
FROM manifest 
WHERE trip_id=73440;

```
### 2. Singleton. Find the trip in which only a single item was transported.
``` sql
+---------+
| trip_id |
+---------+
| 73738   |
+---------+

SELECT trip_id 
FROM manifest 
GROUP BY trip_id 
HAVING COUNT(barcode)=1;
```
### 3. Gavin Brandon. Which company did Gavin Brandon deliver to between the 24th and 25th of April?
``` sql
+--------------+
| company_name |
+--------------+
| Runnel Ltd.  |
+--------------+

SELECT company_name
FROM customer
WHERE reference = (
	SELECT DISTINCT delivery_customer_ref
	FROM manifest
	WHERE trip_id = (
		SELECT trip_id
		FROM trip
		WHERE employee_no = (
			SELECT employee_no
			FROM driver
			WHERE first_name = 'Gavin'
			AND last_name = 'Brandon')
			AND departure_date
			BETWEEN '2012-04-24 00:00:00'
			AND '2012-04-25 00:00:00'));
```
### 4. Long haul. Which driver was responsible for the longest trip?
``` sql
+------------+-----------+------+
| first_name | last_name | days |
+------------+-----------+------+
| Philip     | Slaight   | 11   |
+------------+-----------+------+


SELECT first_name, last_name, DATEDIFF (return_date, departure_date) AS days 
FROM driver 
	JOIN trip ON driver.employee_no=trip.employee_no 
WHERE DATEDIFF(return_date, departure_date)>=
ALL(SELECT DATEDIFF(return_date, departure_date) 
	FROM trip 
	WHERE DATEDIFF(return_date,departure_date)>0);
```
### 5. Peak district. Find the town where we do the most business – ie the one where the largest number of items are picked up and delivered.
``` sql
+-----------+-------+
| town      | Items |
+-----------+-------+
| Gateshead | 328   |
+-----------+-------+

SELECT town, COUNT(trip_id) as 'Items'
FROM customer
INNER JOIN manifest ON customer.reference = manifest.pickup_customer_ref 
OR customer.reference = manifest.delivery_customer_ref 
GROUP BY town
ORDER BY COUNT(*) DESC
LIMIT 1;
```
### 6. Least used. Find the five trucks that are least used during the six months covered by the data. Order by the number of trips they were used on.
``` sql
+--------+--------------+--------------+-------+
| make   | model        | registration | trips |
+--------+--------------+--------------+-------+
| Scania | P94 4x2      | WY51OLV      | 17    |
| DAF    | FTGCF85.460V | PY12 ZYA     | 18    |
| Scania | R270 6x2     | BD08AOG      | 18    |
| DAF    | FTGCF85.460V | PY61 RNV     | 18    |
| DAF    | FTGCF85.460E | PY58 UHF     | 18    |
+--------+--------------+--------------+-------+

SELECT make,model.model,registration,COUNT(*) AS trips 
FROM model 
JOIN vehicle ON model.model=vehicle.model 
JOIN trip ON vehicle.vehicle_id=trip.vehicle_id 
GROUP BY registration,make,model 
ORDER BY COUNT(*) 
LIMIT 5;
```
### 7. Customer satisfaction. Each month the company emails the FOUR customers with the highest number of pickups (not manifest items) to check they are happy with the service. List the top FOUR customers for June.
``` sql
+-----------+------------------------+---------+
| reference | company_name           | Pickups |
+-----------+------------------------+---------+
| 99        | Temerarious & Co       | 9       |
| 3         | Trochiline Services    | 9       |
| 283       | Contemper Retail       | 9       |
| 264       | Byssiferous Industrial | 9       |
+-----------+------------------------+---------+

SELECT customer.reference, customer.company_name, COUNT(DISTINCT trip.trip_id) Pickups
FROM customer
JOIN manifest ON customer.reference = pickup_customer_ref
JOIN trip ON manifest.trip_id = trip.trip_id
WHERE customer.reference = manifest.pickup_customer_ref
    AND manifest.trip_id = trip.trip_id
    AND departure_date
        BETWEEN '2012-06-01' AND '2012-06-30'
GROUP BY company_name, reference
ORDER BY Pickups DESC
LIMIT 4;
```
### 8. Gently does it. Which drivers have never transported anything fragile? (NB Output is abbreviated – in your submission, all 17 rows should be included)
``` sql

+------------+-------------+
| first_name | last_name   |
+------------+-------------+
| Barry      | Thayre      |
| Henry      | Cobelli     |
| Rudyard    | Basillon    |
| Gareth     | Cruickshank |
| ….         | ….          |
| Edgar      | Strank      |
| Nadir      | Millbank    |
| Durant     | Dankersley  |
| Albert     | Phillimore  |
+------------+-------------+

SELECT DISTINCT first_name, last_name
FROM driver 
JOIN trip ON driver.employee_no = trip.employee_no
JOIN manifest ON trip.trip_id = manifest.trip_id 
WHERE driver.employee_no NOT IN ( 
	SELECT employee_no
	FROM trip 
	JOIN manifest ON trip.trip_id = manifest.trip_id 
	AND category = 'B');
```
### 9. Travelling light. Usually, the sequence of pickups and deliveries has to be carefully managed so as not to exceed the vehicle’s capacity at any point. However, if the total weight of manifest items for the whole trip does not exceed the limit, these checks can be skipped. How many trips can proceed without checking?
``` sql
+----------+
| count(*) |
+----------+
| 341      |
+----------+

SELECT count(*)
FROM
(
    SELECT trip.trip_id, SUM(weight) AS weight
    FROM manifest
    JOIN trip
    ON manifest.trip_id = trip.trip_id
    GROUP BY trip.trip_id
) as total
JOIN
(
    SELECT trip.trip_id, ABS(gvw - kerb) AS maximum_weight
    FROM model
    JOIN trip, vehicle
    WHERE trip.vehicle_id = vehicle.vehicle_id
        AND vehicle.model = model.model
    ORDER BY trip_id
) as maximum
WHERE total.trip_id = maximum.trip_id
AND NOT total.weight > maximum.maximum_weight;
```
### 10. Average number of trips. What is the average number of trips per driver in each month? Order the results by month.
``` sql
+------------+-------+
| trip_month | trips |
+------------+-------+
| January    | 3.7   |
| February   | 3.3   |
| March      | 3.6   |
| April      | 3.5   |
| May        | 3.5   |
| June       | 3.5   |
| July       | 1.0   |
+------------+-------+

SELECT drivers.trip_month, FORMAT(total.trips / drivers.trips, 1) as trips
FROM (
    SELECT DISTINCT MONTHNAME(`departure_date`) as trip_month, COUNT(*) as trips
            FROM trip
            JOIN driver
            ON driver.employee_no = trip.employee_no
            GROUP BY trip_month
) as total
JOIN (
    SELECT DISTINCT drivers_per_month.trip_month, COUNT(drivers_per_month.trip_month) as trips
    FROM (
        SELECT DISTINCT MONTHNAME(`departure_date`) as trip_month, driver.employee_no
                FROM trip
                JOIN driver
                ON driver.employee_no = trip.employee_no
    ) as drivers_per_month
    GROUP BY drivers_per_month.trip_month
) as drivers
ON total.trip_month = drivers.trip_month
ORDER BY FIELD(drivers.trip_month,'January','February','March','April','May','June','July');
```
### 11. Dangerous driving. For all trips where hazardous good were transported, find the percentage of each category of item in the manifest. Sort in descending order of the percentage of hazardous items. (NB Output is abbreviated – in your submission, all 48 rows should be included.)
``` sql
+---------+------+------+------+
| trip_id | A    | B    | C    |
+---------+------+------+------+
| 73832   | 44%  | 0%   | 56%  |
| 73404   | 60%  | 0%   | 40%  |
| 73773   | 63%  | 0%   | 38%  |
| 73551   | 64%  | 0%   | 36%  |
| 73013   | 67%  | 0%   | 33%  |
| …       | …    | …    | …    |
| 74059   | 96%  | 0%   | 4%   |
| 73049   | 96%  | 0%   | 4%   |
+---------+------+------+------+

SELECT DISTINCT x.trip_id, 
CONCAT(Round((SELECT COUNT(*) FROM manifest WHERE category = 'A' AND manifest.trip_id = x.trip_id) * 100 / (SELECT COUNT(*) FROM manifest WHERE manifest.trip_id = x.trip_id)), "%") as A, 
CONCAT(Round((SELECT COUNT(*) FROM manifest WHERE category = 'B' AND manifest.trip_id = x.trip_id) * 100 / (SELECT COUNT(*) FROM manifest WHERE manifest.trip_id = x.trip_id)), "%") as B,
CONCAT(Round((SELECT COUNT(*) FROM manifest WHERE category = 'C' AND manifest.trip_id = x.trip_id) * 100 / (SELECT COUNT(*) FROM manifest WHERE manifest.trip_id = x.trip_id)), "%") as C
FROM manifest x
WHERE x.category = 'C'
ORDER BY (SELECT COUNT(*) FROM manifest WHERE category = 'C' AND manifest.trip_id = x.trip_id) * 100 / (SELECT COUNT(*) FROM manifest WHERE manifest.trip_id = x.trip_id)DESC;
```
### 12. Unused trucks. List the registration numbers of the trucks that were not in use between 1 and 5 April inclusive
``` sql
+--------------+
| registration |
+--------------+
| SDU 567M     |
| PY06 BYP     |
| PY61 RNU     |
| BD60BVF      |
| BD08AOF      |
+--------------+

SELECT DISTINCT total_vehicles.registration
FROM
(
    SELECT DISTINCT vehicle.registration
    FROM vehicle
    JOIN trip
    ON trip.vehicle_id = vehicle.vehicle_id
) AS total_vehicles
LEFT JOIN
(
    SELECT DISTINCT vehicle.registration
    FROM vehicle
    JOIN trip t
    ON t.vehicle_id = vehicle.vehicle_id
    WHERE t.departure_date <= DATE('2012-04-01')
        AND t.return_date BETWEEN DATE('2012-04-01') AND DATE('2012-04-05')
        OR t.departure_date <= DATE('2012-04-01') AND t.return_date >= DATE('2012-04-05')
        OR t.departure_date BETWEEN DATE('2012-04-01') AND DATE('2012-04-05')
) AS used_vehicles
ON total_vehicles.registration = used_vehicles.registration
WHERE used_vehicles.registration IS NULL;
```
### 13. Bonus. If a driver works more than 22 days in any one month, they are paid at a higher rate for the extra days. List the drivers who qualify for bonus payments for each month in the data and include the number of extra days worked. Drivers who are not eligible for a bonus should not be shown. Order by month and number of days descending.
``` sql
+----------+-----------------------+------+------------+
| Month    | Name                  | Days | Bonus days |
+----------+-----------------------+------+------------+
| January  | Igor Woodruffe        | 24   | 2          |
| January  | Henry Cobelli         | 23   | 1          |
| February | Tristan Crumbie       | 23   | 1          |
| March    | Oscar Nutten          | 27   | 5          |
| March    | Daniel Miliffe        | 26   | 4          | 
| March    | Eden Blackbrough      | 23   | 1          |
| March    | Henry Cobelli         | 23   | 1          |
| March    | Barry Thayre          | 23   | 1          |
| April    | Durant Kewzick        | 24   | 2          |
| April    | Leonardo Charlet      | 24   | 2          |
| May      | Solomon Alessandrucci | 24   | 2          |
| June     | Lee Rookledge         | 28   | 6          |
| June     | Durant Kewzick        | 26   | 4          |
+----------+-----------------------+------+------------+

SELECT MONTHNAME(departure_date) AS Month,
    CONCAT(d.first_name, ' ', d.last_name) AS Name,
    SUM(DATEDIFF(return_date, departure_date)) AS Days,
    SUM(DATEDIFF(return_date, departure_date)) - 22 AS 'Bonus days'
FROM trip 
JOIN driver d ON d.employee_no = trip.employee_no
GROUP BY Name, Month
HAVING Days > 22
ORDER BY FIELD(MONTH,'January','February','March','April','May','June','July'), Days DESC;

```
### 14. Peak week. Find the busiest week based on the number of departures and returns. Show the start date assuming that a week starts on Monday and the number of departures and returns.
``` sql
+------------+-----------+
| Week       | Movements |
+------------+-----------+
| 2012-04-02 | 105       |
+------------+-----------+

SELECT DATE_ADD(DATE('2011-12-26'), INTERVAL x.departure_week WEEK) AS Week,
    (x.trips + y.trips) AS Movements
FROM (
    SELECT COUNT(dep.departure_week) AS trips,
        dep.departure_week
    FROM (
        SELECT WEEK(departure_date,1) AS departure_week
        FROM trip) AS dep
        GROUP BY dep.departure_week)
AS x
JOIN (
    SELECT COUNT(ret.return_week) AS trips,
        ret.return_week
    FROM (
        SELECT WEEK(return_date,1) AS return_week
        FROM trip) AS ret
        GROUP BY ret.return_week)
AS y
ON departure_week = return_week
ORDER BY Movements DESC
LIMIT 1;
```
### 15. Capacity factor. 100% capacity is when every truck is in use every day. If some trucks are idle, the capacity factor is less than 100%. What is the total capacity factor for the company for the time period covered by the data?
``` sql
+----------+
| capacity |
+----------+
| 42%      |
+----------+

SELECT CONCAT(IFNULL(ROUND(actual.days / possible.maximum_hours * 100), 0), '%') AS capacity
FROM (
    SELECT SUM(a.used) AS days
    FROM (
        SELECT trip_id,
        DATEDIFF(return_date, departure_date) AS used
    FROM trip)
    AS a
) AS actual
INNER JOIN (
    SELECT (total.total_possible * vehicles.vehicles_available) AS maximum_hours
    FROM (
        SELECT DATEDIFF(x.departure_date, y.departure_date) AS total_possible
        FROM (
            SELECT departure_date FROM trip ORDER BY departure_date DESC LIMIT 1
        ) AS x
        INNER JOIN (
            SELECT departure_date FROM trip ORDER BY departure_date ASC LIMIT 1
        ) AS y
    ) AS total
    INNER JOIN (
        SELECT COUNT(*) AS vehicles_available FROM vehicle
    ) AS vehicles
) AS possible;
```

# Section Two
## Modify the diagram and add code to display the added/changed tables

### Adds cam_no to table.
``` sql
ALTER TABLE customer
ADD cam_no INTEGER NOT NULL
AFTER contact_email;
```
### Creates cam table.
``` sql
CREATE TABLE customer_account_manager(
    cam_no INTEGER auto_increment PRIMARY KEY,
    first_name VARCHAR(20) NOT NULL,
    last_name VARCHAR(20) NOT NULL,
    ni_no VARCHAR(13),
    telephone VARCHAR(20),
    mobile VARCHAR(20)
);
```
### Makes cam_no a foreign key.
``` sql
ALTER TABLE customer
ADD FOREIGN KEY customer(cam_no)
REFERENCES customer_account_manager(cam_no);
```
### Creates enquiry table.
``` sql
CREATE TABLE enquiry(
    enquiry_id INTEGER auto_increment PRIMARY KEY,
    date_composed DATE NOT NULL,
	enquiry_topic VARCHAR(20) NOT NULL,
	content TEXT NOT NULL,
    customer_ref INTEGER NOT NULL
);

```
### Creates response table.
``` sql
CREATE TABLE response(
    response_id INTEGER auto_increment PRIMARY KEY,
    date_composed DATE NOT NULL,
    content TEXT NOT NULL,
	enquiry_id INTEGER NOT NULL,
    cam_no INTEGER NOT NULL
);
```
### Makes customer_ref foreign key.
``` sql
ALTER TABLE enquiry
ADD FOREIGN KEY (customer_ref)
REFERENCES customer(reference);
```
### Makes query_id and cam_id foreign key.
``` sql
ALTER TABLE response
ADD FOREIGN KEY (enquiry_id)
REFERENCES enquiry(enquiry_id),
ADD FOREIGN KEY (cam_no)
REFERENCES customer_account_manager(cam_no);
```
### Creates query_details table.
``` sql
CREATE TABLE enquiry_details(
    enquiry_id INTEGER NOT NULL,
	enquiry_condition TEXT NOT NULL,
    response_id INTEGER NOT NULL
);
```
### Makes foreign keys.
``` sql
ALTER TABLE enquiry_details
ADD FOREIGN KEY (enquiry_id)
REFERENCES enquiry(enquiry_id),
ADD FOREIGN KEY (response_id)
REFERENCES response(response_id);
```
### Insert data for cam
``` sql
INSERT INTO customer_account_manager (
	first_name, 
	last_name, 
	ni_no, 
	telephone, 
	mobile)
VALUES(
    'Valeri',
    'Vladimirov',
    'D123456',
    '0123456789',
    '0895 730293');
INSERT INTO customer_account_manager (
	first_name, 
	last_name, 
	ni_no, 
	telephone, 
	mobile)
VALUES( 
    'Petar',
    'Atanasov',
    'G123456',
    '0123456798',
    '0895 730220');
INSERT INTO customer_account_manager (
	first_name, 
	last_name, 
	ni_no, 
	telephone, 
	mobile)
VALUES(
    'Gosho',
    'Mulniqta',
    'P123456',
    '0123456829',
    '0895 223344');

```
### Insert data for customer
``` sql
INSERT INTO customer (
	company_name,
    address,
    town,
    post_code,
    telephone,
    contact_fname,
    contact_sname,
    contact_email,
    cam_no)
VALUES(
    'Tesco',  
    'Bainfield Place',
    'Edinburgh',
    'EH11PAS',
    '0895 882299',
    'Pesho',
    'Mladiq',
    'p.mladiq@gmail.com',
    '2');

INSERT INTO customer (
	company_name,
    address,
    town,
    post_code,
    telephone,
    contact_fname,
    contact_sname,
    contact_email,
    cam_no)
VALUES(
    'Asda',  
    'Orwell Terrace',
    'Edinburgh',
    'EH12AAA',
    '0895 882093',
    'Mariika',
    'Qkata',
    'm.qkata@gmail.com',
    '2');

INSERT INTO customer (
	company_name,
    address,
    town,
    post_code,
    telephone,
    contact_fname,
    contact_sname,
    contact_email,
    cam_no)
VALUES(
    'Lidl',  
    'Slateford Road',
    'Edinburgh',
    'EH11GGG',
    '0895 994582',
    'Viktor',
    'Veliki',
    'v.veliki@gmail.com',
    '2');
```
### Insert data for enquiry
``` sql
INSERT INTO enquiry (
	date_composed, 
	enquiry_topic, 
	customer_ref, 
	content)
VALUES(
    '2012-04-01',
	'Bad service!',
    301,
    'Item delivered was broken ..');
INSERT INTO enquiry (
	date_composed, 
	enquiry_topic, 
	customer_ref, 
	content)
VALUES(
    '2012-05-21',
	'Very impressed!',
    302,
    'Came earlier than expected. Thanks!');
INSERT INTO enquiry (
	date_composed, 
	enquiry_topic, 
	customer_ref, 
	content)
VALUES(
    '2012-07-25',
	'Driver bad mood.',
    303,
    'Next time send a more polite drive. Thanks!');
```
### Insert data for response
``` sql
INSERT INTO response(
    date_composed,
    enquiry_id,
    content,
    cam_no)
VALUES (
    '2012-04-07',
    1,
    'Sorry, we will buy you a new one!',
    2
);

INSERT INTO response(
    date_composed,
    enquiry_id,
    content,
    cam_no)
VALUES (
    '2012-05-26',
    2,
    'When we can we deliver earlier! No Worries.',
    2
);

INSERT INTO response(
    date_composed,
    enquiry_id,
    content,
    cam_no)
VALUES (
    '2012-05-26',
    3,
    'We will fire him!',
    2
);
```
### Insert data for enquiry details
``` sql
INSERT INTO enquiry_details (
	enquiry_id,
	enquiry_condition,
	response_id)
VALUES (
    1,
	'Pending',
    1
);

INSERT INTO enquiry_details(
	enquiry_id,
	enquiry_condition,
	response_id)
VALUES (
    2,
	'Completed',
    2
);

INSERT INTO enquiry_details(
	enquiry_id,
	enquiry_condition,
	response_id)
VALUES (
    3,
	'Completed',
    3
);
```
