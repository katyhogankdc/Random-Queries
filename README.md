# Random-Queries

```
--Attendance by Membership - Count of Absent, Tardy, Present 

SELECT 

 e.grade_level AS GradeLevel
, e.schoolid
--, cd.date_value
--, dsc.schoolyearshort AS SchoolYearShort
, cd.date_value AS FullDate
, sum(cd.membershipvalue) AS Membership,
--, COALESCE (ac.att_code, 'P') AttendanceCode, 
	SUM (CASE 
		WHEN ac.att_code = 'T' THEN 1
		WHEN ac.att_code = 'TE' THEN 1
		WHEN ac.att_code = 'TC' THEN 1
		WHEN ac.att_code = 'TRE' THEN 1
	ELSE 0 
	END) AS TardyCount, 
	SUM (CASE 
		WHEN ac.presence_status_cd = 'Absent' THEN 1
	ELSE 0 
	END) AS AbsenceCount, 
	SUM(CASE
		WHEN ac.att_code IS NULL THEN 1
		WHEN ac.att_code = 'ISS' THEN 1
		WHEN ac.att_code = 'NA' THEN 1
		WHEN ac.att_code = 'R' THEN 1
	ELSE 0
	END) AS PresentCount
--, COALESCE(ac.description,'Present') AttendanceDescription
--, COALESCE(ac.presence_status_cd,'Present') PresentStatus


FROM  (
 SELECT s.student_number, s.id studentid, s.lastfirst, s.last_name, s.first_name, s.schoolid, s.grade_level, s.entrydate, s.exitdate, s.exitcode, 'Students' AS 'EnrollmentSource'
 FROM [powerschool].[PowerSchool_STUDENTS] s

 UNION

 SELECT s.student_number, r.studentid, s.lastfirst, s.last_name, s.first_name, r.schoolid, r.grade_level, r.entrydate, r.exitdate, r.exitcode, 'Reenrollments' AS 'EnrollmentSource'
 FROM [powerschool].[PowerSchool_REENROLLMENTS] r
 INNER JOIN [powerschool].[PowerSchool_STUDENTS] s ON s.id = r.studentid
) e 
INNER JOIN [powerschool].[PowerSchool_CALENDAR_DAY] cd ON (cd.date_value BETWEEN e.entrydate AND e.exitdate-1) AND e.schoolid = cd.schoolid
LEFT JOIN [powerschool].[PowerSchool_ATTENDANCE] att ON (att.studentid = e.studentid AND att.att_date = cd.date_value AND att.att_mode_code = 'ATT_ModeDaily')
LEFT JOIN [powerschool].[PowerSchool_ATTENDANCE_CODE] ac ON ac.id = att.attendance_codeid
INNER JOIN [powerschool].[PowerSchool_SCHOOLS] sc ON sc.school_number = e.schoolid
INNER JOIN [powerschool].[PowerSchool_STUDENTS] s ON s.id = e.studentid
INNER JOIN [custom].[custom_StudentBridge] sb ON sb.student_number = s.student_number
INNER JOIN [custom].[custom_Students] cs ON cs.systemstudentid = sb.systemstudentid
INNER JOIN [dw].[DW_dimSchoolCalendar] dsc ON (dsc.systemschoolid = CONVERT(varchar,e.schoolid)  AND dsc.fulldate = cd.date_value)


WHERE 1=1
AND cd.insession = 1
AND ((e.exitcode != '4321' AND e.exitcode != '1234') OR e.exitcode IS NULL)
--AND e.schoolid = 1100
AND ((cd.date_value BETWEEN '2016-08-08' AND '2017-06-20'))

GROUP BY e.schoolid, e.grade_level, ac.att_code, ac.presence_status_cd, cd.date_value

order by cd.date_value, e.schoolid, e.grade_level

```
```
--Attendance and Enrollment by Homeroom
select 
a.schoolid,
a.att_date,
ac.att_code,
--s.student_number,
s.grade_level,
SUM(ac.calculate_ADA_YN) AS Attendance,
SUM(ac.calculate_ADM_YN) AS Enrollment,
s.home_room
from powerschool.powerschool_attendance a
left join powerschool.powerschool_attendance_code ac on ac.schoolid = a.schoolid and ac.id = a.attendance_codeid
left join powerschool.powerschool_students s on s.id = a.studentid
where a.att_mode_code = 'ATT_ModeDaily'
and s.enroll_status = 0
and a.yearid = 27
and a.att_date > '2017-08-01'

group by a.schoolid, a.att_date, ac.att_code, s.grade_level, s.home_room
```
```
--Meal Program by Homeroom
select 
rm.entry_date, 
rm.home_room AS Homeroom,
rm.meal_count as MealCount, 
rm.meal_type AS MealType,
sc.abbreviation AS SchoolName
from u_record_meals rm
JOIN schools sc on sc.school_number = rm.school_id
where rm.entry_date > '01-AUG-17'
order by rm.entry_date, sc.abbreviation, rm.home_room
```

```
-----Student Retention Information
SELECT
lastfirst,
student_number,
course_number,
schoolid,
count(*)
FROM (select distinct
s.student_number,
s.lastfirst,
t.yearid,
cc.course_number,
s.schoolid
from cc
join terms t on t.id = cc.termid 
join students s on s.id = cc.studentid
where cc.course_number LIKE '%02' and s.enroll_status = 0
AND (LENGTH(CC.COURSE_NUMBER) =3 OR CC.COURSE_NUMBER = 'PK02' OR CC.COURSE_NUMBER = 'PS02')
AND (CC.DATELEFT - CC.DATEENROLLED) > 30
ORDER BY S.STUDENT_NUMBER) sbq
group by student_number, lastfirst, course_number, schoolid
having count(*) > 1
ORDER BY SCHOOLID;
```

```
-------Checking Discrepancies in Meal Program based on Updated Date
select 
rm.entry_date, 
rm.whencreated,
--rm.home_room AS Homeroom,
SUM(rm.meal_count) as MealCount, 
rm.meal_type AS MealType,
sc.abbreviation AS SchoolName
from u_record_meals rm
JOIN schools sc on sc.school_number = rm.school_id
where rm.entry_date between '12-feb-18' and '26-feb-18'
and sc.abbreviation = 'Quest'
group by sc.abbreviation, rm.meal_type, rm.entry_date, rm.whencreated
order by rm.entry_date, sc.abbreviation;
```

```
--------Aggregated Meal Data 
select 
rm.entry_date, 
--rm.home_room AS Homeroom,
SUM(rm.meal_count) as MealCount, 
rm.meal_type AS MealType,
sc.abbreviation AS SchoolName
from u_record_meals rm
JOIN schools sc on sc.school_number = rm.school_id
where rm.entry_date >= '03-AUG-17'
group by sc.abbreviation, rm.meal_type, rm.entry_date
order by rm.entry_date, sc.abbreviation;
```

```
--------------Matching Siblings
select
p.lastfirst as StudentName,
p.student_number as StudentNumber,
p.schoolid AS StudentSchool,
sib.lastfirst AS SiblingName,
sib.student_number AS SiblingStudentNumber,
sib.schoolid AS SiblingSchool,
p.relationship,
p.name,
p.type
from
    ((select studentsdcid,
    parent1_relationship AS Relationship,
    CONCAT(parent1_last, parent1_first) AS Name,
    lastfirst, 
    student_number,
    enroll_status,
    schoolid,
    'Parent 1' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
    UNION 
    (select studentsdcid, 
    parent2_relationship AS Relationship, 
    CONCAT(parent2_last, parent2_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 2' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent3_relationship AS Relationship, 
    CONCAT(parent3_last, parent3_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 3' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent4_relationship AS Relationship, 
    CONCAT(parent4_last, parent4_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 4' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent5_relationship AS Relationship, 
    CONCAT(parent5_last, parent5_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,   
    'Parent 5' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent6_relationship AS Relationship, 
    CONCAT(parent6_last, parent6_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 6' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent7_relationship AS Relationship, 
    CONCAT(parent7_last, parent7_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 7' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent8_relationship AS Relationship, 
    CONCAT(parent8_last, parent8_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 8' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent9_relationship AS Relationship, 
    CONCAT(parent9_last, parent9_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 9' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent10_relationship AS Relationship, 
    CONCAT(parent10_last, parent10_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 10' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid))p
join 
    ((select studentsdcid,
    parent1_relationship AS Relationship,
    CONCAT(parent1_last, parent1_first) AS Name,
    lastfirst, 
    student_number,
    enroll_status,
    schoolid,
    'Parent 1' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
    UNION 
    (select studentsdcid, 
    parent2_relationship AS Relationship, 
    CONCAT(parent2_last, parent2_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 2' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent3_relationship AS Relationship, 
    CONCAT(parent3_last, parent3_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 3' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent4_relationship AS Relationship, 
    CONCAT(parent4_last, parent4_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 4' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent5_relationship AS Relationship, 
    CONCAT(parent5_last, parent5_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 5' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent6_relationship AS Relationship, 
    CONCAT(parent6_last, parent6_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 6' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent7_relationship AS Relationship, 
    CONCAT(parent7_last, parent7_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 7' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent8_relationship AS Relationship, 
    CONCAT(parent8_last, parent8_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 8' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent9_relationship AS Relationship, 
    CONCAT(parent9_last, parent9_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 9' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid)
UNION
    (select studentsdcid, 
    parent10_relationship AS Relationship, 
    CONCAT(parent10_last, parent10_first) AS Name,
    lastfirst,
    student_number,
    enroll_status,
    schoolid,
    'Parent 10' AS Type
    from u_students us
    join students s on s.dcid = us.studentsdcid))sib on p.name = sib.name --and p.relationship = sib.relationship
where p.enroll_status = 0
and sib.enroll_status = 0
and p.student_number != sib.student_number 
and p.name != '(null)'
and sib.name != '(null)'
order by p.lastfirst; 
```

