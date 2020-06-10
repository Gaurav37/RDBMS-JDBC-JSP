# DBMS application development using JDBC and JSP

This was a recent project to create an app which can be utilized by people moving to a new area for checking the best available medical centers or facilities in their area.

Technologies used for this project are Java, MySQL, JDBC, JSP and tools used were excel, MySQL workbench, Eclipse, Clover(for extraction, transformation and loading of data from 2nd source)

First step was designing UML diagram after which database was designed using tables, views, triggers, procedures, event scheduler etc. Then various JDBC classes and JSP classes were made to make it work as an application.

Let us take a brief look at some part of the whole code which can help us understand the overall workflow of the application.

```
LOAD DATA INFILE 'Inpatient_Rehab_Ratings.csv'
 INTO TABLE RehabilitationRating
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\r\n'
IGNORE 1 LINES
(@dummy, CMSId, @dummy, @dummy, @dummy, @dummy, @dummy, @dummy, @dummy, @dummy,
 @dummy, @dummy, @StartString, @EndString, @UlcerVar, @FluVaccVar, @UTIVar, @GoalVar, @FallVar, @CDIVar,
 @FluCoverVar, @Readmiss30Var, @ReadmissStayVar, @ReturnHomeVar, @SpendingVar)
 Set StartDate = str_to_date(@StartString, '%m/%d/%Y'),
 EndDate = str_to_date(@EndString, '%m/%d/%Y'),
 UlcerRate= nullif(@UlcerVar,'NA'),
 FluVaccRate = nullif(@FluVaccVar,'NA'),
 UTIRate = nullif(@UTIVar,'NA'),
 AssessGoalRate = nullif(@GoalVar,'NA'),
 FallRate = nullif(@FallVar,'NA'),
 CDIRate = nullif(@CDIVar,'NA'),
 FluCoverStaff = nullif(@FluCoverVar,'NA'),
 ReadmissRate30Days = nullif(@Readmiss30Var,'NA'),
 ReadmissRateDuringStay = nullif(@ReadmiisStayVar,'NA'),
 ReturnHomeRate = nullif(@ReturnHomeVar,'NA'),
 MedicareSpendingPerBenificiary = nullif(@SpendingVar, 'NA');
```
Above code is for loading data from CSV files in MySQL. Notice the use of str_to_date(for converting string date to date type), ENCLOSED BY/OPTIONALLY ENCLOSED BY, IGNORE 1 LINES(for ignoring header of csv file. Omit this if you don't have header in your source csv file).

```
drop procedure if exists uspCreateView;
delimiter //
CREATE PROCEDURE uspCreateView()
begin
declare limValue int;
set limValue=100;
create or replace view view1 as select LL.StateName, count(LL.StateName) as Count, avg(R.OverallRating) as PatNursinghomeRating, 
avg(HR.Overall_patient_rating) as PathealthcareRating, avg(HH.OverallRating) as OverallHospitalRating , 
(avg(R.OverallRating)+avg(HR.Overall_patient_rating)+avg(HH.OverallRating))/3 as avgOverall
from NursingHome N, NursingHomeRating R, HomeHealthcare H, HomeHealthcareRating HR, ZipsInLocations ZL,Locations LL, 
Hospitals HH
 where N.ProviderID=R.ProviderID and H.CMSId=HR.CMSId and N.ZipCode=H.ZipCode and H.ZipCode=ZL.ZIpCode and HH.ZipCode=H.ZipCode and 
 ZL.LocationID= LL.LocationID
 and R.OverallRating is not null and Overall_patient_rating is not null and HH.OverallRating is not null 
 and ABS(R.OverallRating-Overall_patient_rating)<2 and ABS(HH.OverallRating-Overall_patient_rating)<2
 and R.OverallRating>=4 and Overall_patient_rating>=4 and HH.OverallRating>=4
 group by StateName
 having avg(R.OverallRating)>=4 and avg(HR.Overall_patient_rating)>=4 and avg(HH.OverallRating)>=4
 order by avgOverall desc;
end//
 delimiter ;
```
Above procedure is to create a new view or replace existing view which shows list of states with average rating and other ratings for different medical institutions such as nursing homes, hospitals and home-healthcare etc. This procedure is called every 24 hours with help of an event scheduler.
```
 SET GLOBAL event_scheduler = ON;
drop event if exists StateHealthcareRating;
CREATE EVENT StateHealthcareRating ON SCHEDULE
    EVERY 1 DAY
    STARTS (TIMESTAMP(CURRENT_DATE) + INTERVAL 1 DAY) ON COMPLETION PRESERVE ENABLE 
  DO
    CALL uspCreateView();
```
By using "on completion preserve" clause in above event scheduling code, we are making sure that the event is not dropped automatically after event is executed because we want to execute the event again and again with an interval of 1 day.
```
drop trigger if exists HomeHealthcareBackupTrigger;
delimiter //
CREATE TRIGGER HomeHealthcareBackupTrigger BEFORE DELETE on HomeHealthcare
FOR EACH ROW
BEGIN
insert into HomeHealthcareBackup(CMSID ,ProviderName ,Address ,City ,
ZipCode ,Phone,NursingCare ,PhysicalTherapy ,OccupationalTherapy ,SpeechPathology ,MedicalSocial ,
HomeHealthAide ,OwnershipType )  (select CMSID ,ProviderName,Address ,City ,ZipCode,Phone ,NursingCare ,PhysicalTherapy ,OccupationalTherapy ,
SpeechPathology ,MedicalSocial ,HomeHealthAide ,OwnershipType from HomeHealthcare where cmsid=old.cmsid);
END//
delimiter ;
```
Trigger like above if created in the database ensure that the row is inserted in backup table before deletion. Keeping it before deletion is a good practice.

Now let us take a look at JDBC and JSP part of code.

Below is a part of connection manager class.
```
public class ConnectionManager {

	private final String user = "root1";
	private final String password = "MysqlDatabase";
	private final String hostName = "localhost";
	private final int port= 3306;
	private final String schema = "HealthCare";

	public Connection getConnection() throws SQLException {
		Connection connection = null;
		try {
			Properties connectionProperties = new Properties();
			connectionProperties.put("user", this.user);
			connectionProperties.put("password", this.password);
			
			try {
				Class.forName("com.mysql.jdbc.Driver");
			} catch (ClassNotFoundException e) {
				e.printStackTrace();
				throw new SQLException(e);
			}
			connection = DriverManager.getConnection(
			    "jdbc:mysql://" + this.hostName + ":" + this.port + "/" + this.schema + "?autoReconnect=true&useSSL=false",
			    connectionProperties);
		} catch (SQLException e) {
			e.printStackTrace();
			throw e;
		}
		return connection;
```
There are 4 variables defined which are username, password, hostname, port, schema. There are two methods getConnection and closeConnection. In getConnection Class.forName("com.mysql.jdbc.Driver") tries to calls jdbc driver where we try to insert username and password(already defined) through properties class which is inherent in connection manager.
