# DBMS application development using JDBC and JSP

This was a recent project to create an app which can be utilized by people moving to a new area for checking the best available medical centers or facilities in their area.

Technologies used for this project are Java, MySQL, JDBC, JSP and tools used were excel, MySQL workbench, Eclipse, Clover(for extraction, transformation and loading of data from 2nd source).

In Eclipse, right click on the package name and select 'Run As' then select 'Run On Server' to run application though Apache Tomcat server using JSP files.
Following are some screenshots from the developed application. 
First picture is main page.

![](RDBMS-JDBC-JSP/Application page 1.png)

## RDBMS Script
First step was designing UML diagram after which database was designed using tables, views, triggers, procedures, event scheduler etc. Then various JDBC classes and JSP classes were made to make it work as an application.

Let us take a brief look at some part of the whole code which can help us understand the overall workflow of the application.

### Load Statements
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

### Procedure Used
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

### Event Schedule
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

### Trigger Used
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

## JDBC and JSP
Now let us take a look at JDBC and JSP part of code.
JDBC part consist of "Data Application Layer", "modeling layer" and inserter file which can be omitted if we want abstraction from user and high user friendliness
JSP part of application covers the view of application from userend; consisting of "servlet files" and webcontents jsp files.

### Connection Manager
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

### Data Application Layer
Following is an example of one class of DAL. After connection instance, this class defines methods for getting data from database. This class's method getBestRatedHealthcareStates() will output a list which will appear as a table. The appearance part is covered by jsp files.
In the method, connection.prepareStatement() prepares the SQL query which is executed using executeQuery(). While(results.next()) stores the result from SQL statement execution into class variables.
These variables are used to create this class's object through which the servlet files will take the data to jsp file display.
```
public class OverallRatingDao {
	protected ConnectionManager connectionManager;

	private static OverallRatingDao instance = null;
	protected OverallRatingDao() {
		connectionManager = new ConnectionManager();
	}
	public static OverallRatingDao getInstance() {
		if(instance == null) {
			instance = new OverallRatingDao();
		}
		return instance;
	}

public List<OverallRating> getBestRatedHealthcareStates() throws SQLException {
	List<OverallRating> overallRating = new ArrayList<OverallRating>();
	String selectOverallRating = "select * from view1;";
	Connection connection = null;
	PreparedStatement selectStmt = null;
	ResultSet results = null;
	try {
		connection = connectionManager.getConnection();
		selectStmt = connection.prepareStatement(selectOverallRating);
		results = selectStmt.executeQuery();
		while(results.next()) {
			String StateName1 = results.getString("StateName");
			int count = results.getInt("count");
			double PatNursinghomeRating1 = results.getDouble("PatNursinghomeRating");
			double PathealthcareRating1 = results.getDouble("PathealthcareRating");
			double Overallhospitalrating1 = results.getDouble("Overallhospitalrating");
			double avgOverall1 = results.getDouble("avgOverall");
			System.out.println(StateName1);
			OverallRating overallRating1 = new OverallRating(StateName1,count,PatNursinghomeRating1,PathealthcareRating1,Overallhospitalrating1,avgOverall1);
			overallRating.add(overallRating1);
		}
	} catch (SQLException e) {
		e.printStackTrace();
		throw e;
	} finally {
		if(connection != null) {
			connection.close();
		}
		if(selectStmt != null) {
			selectStmt.close();
		}
		if(results != null) {
			results.close();
		}
	}
	return overallRating;
}
}
```
### Model Files
This constructor in this class is used by DAL layer classes to make an object containing all the query data and also the methods defined in this class are used by JSP files through servlet files to display information. This class contains a constructor of variables corresponding to the table columns in database and getters and setters methods to output those variables data and to store the input data if at all to the database.
Any changes in the DAL layer class variables should be reflected in this class also. 
```
public class OverallRating {
	protected String StateName;
	protected int count;
	protected double PatNursinghomeRating;
	protected double PathealthcareRating;
	protected double Overallhospitalrating;
	protected double AvgOverall;
	
	public OverallRating(String stateName, int count, double patNursinghomeRating,
			double pathealthcareRating, double overallhospitalrating, double avgOverall) {
		
		StateName = stateName;
		this.count = count;
		PatNursinghomeRating = patNursinghomeRating;
		PathealthcareRating = pathealthcareRating;
		Overallhospitalrating = overallhospitalrating;
		AvgOverall=avgOverall;
	}
	public double getAvgOverall() {
		return AvgOverall;
	}
	public void setAvgOverall(double avgOverall) {
		AvgOverall = avgOverall;
	}
	public String getStateName() {
		return StateName;
	}
	public void setStateName(String stateName) {
		StateName = stateName;
	}
	public int getCount() {
		return count;
	}
	public void setCount(int count) {
		this.count = count;
	}
	public double getPatNursinghomeRating() {
		return PatNursinghomeRating;
	}
	public void setPatNursinghomeRating(double patNursinghomeRating) {
		PatNursinghomeRating = patNursinghomeRating;
	}
	public double getPathealthcareRating() {
		return PathealthcareRating;
	}
	public void setPathealthcareRating(double pathealthcareRating) {
		PathealthcareRating = pathealthcareRating;
	}
	public double getOverallhospitalrating() {
		return Overallhospitalrating;
	}
	public void setOverallhospitalrating(double overallhospitalrating) {
		Overallhospitalrating = overallhospitalrating;
	}
}
```
### Servlet File
Servlet is a server-side program that handles client requests through frontend jsp file and implements the servlet interface. Tries to fetch the data from model file, as requested by JSP. 
```
@WebServlet("/findOverallRating")
public class FindOverallRating extends HttpServlet {
	
	protected OverallRatingDao overallRatingDao;
	
	@Override
	public void init() throws ServletException {
		overallRatingDao = OverallRatingDao.getInstance();
	}
	
	@Override
	public void doGet(HttpServletRequest req, HttpServletResponse resp)
			throws ServletException, IOException {
		// Map for storing messages.
		Map<String, String> messages = new HashMap<String, String>();
        req.setAttribute("messages", messages);

        List<OverallRating> overallRating = new ArrayList<OverallRating>();
        
        try {
        		overallRating = overallRatingDao.getBestRatedHealthcareStates();
        		System.out.println(overallRating.get(0));
            } catch (SQLException e) {
    			e.printStackTrace();
    			throw new IOException(e);
            }
        	messages.put("success", "Displaying top average healthcare ratings for states");
        
        req.setAttribute("overallratings", overallRating);
        
        req.getRequestDispatcher("/FindOverallRating.jsp").forward(req, resp);
	}
}
```
### JSP file 
JSP file is used to provide frontend to user at the same time to communicate with the server using other files discussed above.
Below is an example of jsp file as used in this project.
Fist few lines are to include libraries, then inside body, a table structure has been defined. Notice the use of foreach to act as a loop for all values in a row and $ prefixed words are variables to store output values to be displayed.
```
<%@ taglib uri="http://java.sun.com/jsp/jstl/functions" prefix="fn" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/fmt" prefix="fmt" %>
<%@ page language="java" contentType="text/html; charset=ISO-8859-1"
    pageEncoding="ISO-8859-1"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="ISO-8859-1">
<title>OverallRatings</title>
</head>
<body>
<h1>Overall healthcare system ratings updated once every 24 hrs</h1>
	<table border="1">
            <tr>
                <th>StateName</th>
                <th>Count</th>
                <th>patient Nursinghome Rating</th>
                <th>Patient healthcare Rating</th>
                <th>Overall Hospital Rating</th>
                <th>Average of all previous Ratings</th>
                
            </tr>
            <c:forEach items="${overallratings}" var="overallrating" >
                <tr>
                    <td><c:out value="${overallrating.getStateName()}" /></td>
                    <td><c:out value="${overallrating.getCount()}" /></td>
                    <td><c:out value="${overallrating.getPatNursinghomeRating()}" /></td>
                    <td><c:out value="${overallrating.getPathealthcareRating()}" /></td>
                    <td><c:out value="${overallrating.getOverallhospitalrating()}" /></td>
                    <td><c:out value="${overallrating.getAvgOverall()}" /></td>
                    
                </tr>
            </c:forEach>
       </table>
  
</body>
</html>
```
