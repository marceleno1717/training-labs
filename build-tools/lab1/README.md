# Lab 1
### 1) Clone the repo
```
git clone https://github.com/Ibrahim-Adel15/CalculatorMavenApp.git
```

### 2) run unit test
```
mvn test
```
Output
```
[INFO] Scanning for projects...
[INFO]
[INFO] -----------------------< com.example:calculator >-----------------------
[INFO] Building calculator 1.0.0
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ calculator ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /home/marceleno/ivolve-labs/build-tools/lab1/CalculatorMavenApp/src/main/resources
[INFO]
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ calculator ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ calculator ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /home/marceleno/ivolve-labs/build-tools/lab1/CalculatorMavenApp/src/test/resources
[INFO]
[INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ calculator ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- maven-surefire-plugin:3.2.5:test (default-test) @ calculator ---
[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider
[INFO]
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.example.calculator.CalculatorTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.114 s -- in com.example.calculator.CalculatorTest
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  2.794 s
[INFO] Finished at: 2026-10-20T15:44:05+03:00
[INFO] ------------------------------------------------------------------------
```
### 3) Build the app
```
mvn package
```
```
[INFO] Scanning for projects...
[INFO]
[INFO] -----------------------< com.example:calculator >-----------------------
[INFO] Building calculator 1.0.0
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ calculator ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /home/marceleno/ivolve-labs/build-tools/lab1/CalculatorMavenApp/src/main/resources
[INFO]
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ calculator ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ calculator ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /home/marceleno/ivolve-labs/build-tools/lab1/CalculatorMavenApp/src/test/resources
[INFO]
[INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ calculator ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- maven-surefire-plugin:3.2.5:test (default-test) @ calculator ---
[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider
[INFO]
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.example.calculator.CalculatorTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.106 s -- in com.example.calculator.CalculatorTest
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO]
[INFO] --- maven-jar-plugin:3.4.1:jar (default-jar) @ calculator ---
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  4.421 s
[INFO] Finished at: 2026-10-20T15:54:07+03:00
[INFO] ------------------------------------------------------------------------
```

### 4) Run the jar file
```
java -jar target/calculator.jar
```
```
Enter first number: 1
Enter second number: 2
a = 1.0, b = 2.0
Sum: 3.0
Subtract: -1.0
Multiply: 2.0
Divide: 0.5
```