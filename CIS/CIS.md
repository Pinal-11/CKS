CIS -> Center for Internet Security
It covers the below vendors.
* OS -> Linux, Windows, MacOS, and more
* Cloud -> Google, AWS, Azure, and more
* Mobile -> Android and iOS
* Network -> CheckPoint, Cisco, Juniper, Palo Alto Networks
* Desktop Soft -> web browsers, MS Offices, Zoom, Etc...
* Server Soft -> Tomcat, Nginx, Docker, Kubernetes, More...


--------------------------------------------------------------------------------------

         Welcome to CIS-CAT Pro Assessor; built on 02/23/2026 22:23 PM
--------------------------------------------------------------------------------------
  This is the Center for Internet Security Configuration Assessment Tool, v4.60.0
          At any time during the selection process, enter 'q!' to exit.
--------------------------------------------------------------------------------------

Verifying application

usage: Assessor-CLI.[bat|sh] -[options] <extras>
---------------------------------------------------------------------------------------------------
 Options                                           Tip
---------------------------------------------------------------------------------------------------
 -b,--benchmark <BMK-OR-DSC>                       Path to file containing assessment content, such
                                                   as an XCCDF or DataStream Collection
 -bi,--benchmark-info                              When used with -b, display information about the
                                                   selected benchmark or datastream collection, such
                                                   as a profile listing
 -cfg,--config-xml <CONFIGURATION XML FILE>        Path to an XML configuration file detailing
                                                   assessments to perform against various sessions.
 -cl,--checklist <CHECKLIST>                       Used only when the -b argument selects a
                                                   <data-stream-collection> document for assessment,
                                                   the -cl option selects the ID of the specific
                                                   checklist to assess
 -csv                                              Generate report results in CSV format
 -D <property=value>                               Customize user properties or interactive values
 -dm,--data-stream <DATA-STREAM>                   Used only when the -b argument selects a
                                                   <data-stream-collection> document for assessment,
                                                   the -dm option selects the ID of the specific
                                                   data-stream to assess
 -e,--encrypt <FILE TO ENCRYPT>                    Full path to the a sessions.properties file or
                                                   configuration.xml file to be encrypted, including
                                                   the file name and its extension
 -ep,--encryption-password <ENCRYPTION PASSWORD>   The password that will be used to encrypt a
                                                   sessions.properties file or configuration.xml
                                                   file
 -fp,--file-password <FILE PASSWORD>               The password originally used to encrypt a
                                                   sessions.properties file or configuration.xml
                                                   file
 -gui                                              The assessment is run by the GUI
 -h,--help                                         Show usage information
 -html                                             Generate report results in HTML format
 -i,--interactive                                  Proceed interactively
 -json                                             Generate report results in JSON format
 -l,--list                                         List available assessment content
 -lv,--list-verbose                                Add verbose output when listing available content
 -narf,--no-arf                                    Do not generate report results in Asset Reporting
                                                   Format
 -nl,--no-logging                                  Disable all logging (Default log level is WARN)
 -npr                                              Generate a JSON report showing results that did
                                                   not pass (fail, error, unknown)
 -nrf,--no-report-file                             Do not generate an assessment report file.  This
                                                   option is intended to be used in conjunction with
                                                   the -u option.  If the -u option is not
                                                   specified, the assessment results will be saved
                                                   as a file regardless.
 -nts,--no-timestamp                               Do not include the auto-generated timestamp as
                                                   part of the report name.
 -o,--definitions                                  Definitions/Vulnerability Assessment
 -od,--oval-definitions <OVAL DEFINITIONS>         Path to file containing OVAL Definitions
 -ov,--oval-variables <OVAL VARIABLES>             Path to file containing OVAL Variables
 -p,--profile <PROFILE>                            ID/Name of the specific profile to assess
 -props,--properties <PROPERTIES-FILE>             Location (absolute or relative to starting
                                                   directory) of custom Assessor properties file
 -q,--quiet                                        Quiet Mode: Disable assessment status output
 -rd,--reports-dir <REPORTS-DIR>                   Path to a directory specifying the location to
                                                   which output reports are saved
 -rp,--report-prefix <REPORT-PREFIX>               Override the default report name.  Timestamp
                                                   information will be appended to the report
                                                   prefix, except when the -nts option is included
                                                   as well.
 -sessions,--sessions <SESSIONS.PROPERTIES>        Location (absolute or relative to starting
                                                   directory) of Sessions configuration file
 -sv                                               Schematron Validation for all Benchmarks. Can
                                                   only run along with -bi
 -test,--test                                      Used in conjunction with the 'sessions' option,
                                                   or with the default 'sessions.properties' file,
                                                   test the validity of session configurations,
                                                   reporting success or failure
 -txt                                              Generate report results in plain-text format
 -u,--url <REPORTS-URL>                            Sends a HTTP POST with the Assessment Results XML
                                                   to the specified URL
 -ui,--ignore-warnings                             Ignore certificate warnings when POSTing results
                                                   to a URL.
 -v,--error                                        Configure log level to ERROR
 -vv,--warn                                        Configure log level to WARN
 -vvv,--info                                       Configure log level to INFO
 -vvvv,--debug                                     Configure log level to DEBUG
 -vvvvv,--trace                                    Configure log level to TRACE
 -vvvvvv,--all                                     Configure log level to ALL
Exit Code: 0
Exit Description: CIS-CAT Pro Assessor Exited Successfully.



controlplane ~ ➜  cat /root/Assessor/Assessor-CLI.sh
#!/bin/sh

# Absolute path to this script, e.g. /home/user/bin/foo.sh
SCRIPT=$(readlink -f "$0")
# Absolute path this script is in, thus /home/user/bin
SCRIPTPATH=$(dirname "$SCRIPT")

JAVA=java
MAX_RAM_IN_MB=2048
DEBUG=0

which $JAVA 2>&1 > /dev/null

if [ $? -ne "0" ]; then
        echo "Error: Java is not in the system PATH."
        exit 1
fi

JAVA_VERSION_RAW=`$JAVA -version 2>&1`

echo $JAVA_VERSION_RAW | grep 'version\s*\"\(\(1\.8\.\)\|\(9\.\)\|\([1-9][0-9]\.\)\)' 2>&1 > /dev/null

if [ $? -eq "1" ]; then

        echo "Error: The version of Java you are attempting to use is not compatible with CISCAT:"
        echo ""
        echo $JAVA_VERSION_RAW
        echo ""
        echo "You must use Java 1.8.x, or higher. The most recent version of Java is recommended."
        exit 1;
fi

if [ $DEBUG -eq "1" ]; then
        echo "Executing CIS-CAT Pro Assessor from $SCRIPTPATH"
        $JAVA -Xmx${MAX_RAM_IN_MB}M -jar $SCRIPTPATH/Assessor-CLI.jar "$@" --verbose
else
        $JAVA -Xmx${MAX_RAM_IN_MB}M -jar $SCRIPTPATH/Assessor-CLI.jar "$@"
fi

controlplane ~ ➜  