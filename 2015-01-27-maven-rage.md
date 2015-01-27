# Custom maven targets from within maven without custom plugin

Sometimes you just want to compose a couple of steps differently for different scenarios.

Say we want to invoke:
  
  mvn package cargo:redeploy integration-test
  
Then we could write a `Makefile`, o we can use the jetspeed:mvn plugin:

  <project>
    <build>
      <plugins>
        <!-- -->
        <plugin>
          <groupId>org.apache.portals.jetspeed-2</groupId>
          <artifactId>jetspeed-mvn-maven-plugin</artifactId>
          <version>2.2.2</version>
          <configuration>
            <targets combine.children="append">
              <target>
                <id>fast-test</id>
                <dir>@rootdir@</dir>
                <goals>package,cargo:redeploy,integration-test</goals>
              </target>

            </targets>
          </configuration>
        </plugin>
      </plugins>
    </build>
  </project>
