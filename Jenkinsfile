node('slave001') {


    stage('Prepare') {
        echo "1.Prepare Stage"
        checkout scm
        //要部署的模块，若是根项目，用点 .
        project_module = '.'
        pom = readMavenPom file: "${project_module}/pom.xml"
        echo "group: ${pom.groupId}, artifactId: ${pom.artifactId}, version: ${pom.version}"
        script {
            build_tag = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
            if (env.BRANCH_NAME != 'master' && env.BRANCH_NAME != null) {
                build_tag = "${env.BRANCH_NAME}-${build_tag}"
            }

            currentBuild.displayName = BUILD_NUMBER + "_" +build_tag
        }
    }

    stage('Compile And UnitTest') {
        echo "2.Compile the code"
        sh "mvn clean install"
        junit testResults: '**/target/*-reports/TEST-*.xml'
    }


    stage('Basic Quality Report') {
        echo "3.Basic quality report"
        sh "mvn site"

        def java = scanForIssues tool: java()
        def javadoc = scanForIssues tool: javaDoc()

        publishIssues id: 'analysis-java', name: 'Java Issues', issues: [java, javadoc]  //, filters: [includePackage('io.jenkins.plugins.analysis.*')]

        def checkstyle = scanForIssues tool: checkStyle(pattern: '**/target/checkstyle-result.xml')
        publishIssues issues: [checkstyle]

        def pmd = scanForIssues tool: pmdParser(pattern: '**/target/pmd.xml')
        publishIssues issues: [pmd]

        def cpd = scanForIssues tool: cpd(pattern: '**/target/cpd.xml')
        publishIssues issues: [cpd]

        def spotbugs = scanForIssues tool: spotBugs(pattern: '**/target/findbugsXml.xml')
        publishIssues issues: [spotbugs]

        def maven = scanForIssues tool: mavenConsole()
        publishIssues issues: [maven]

        publishIssues id: 'analysis', name: 'All Issues',
                issues: [checkstyle, pmd, spotbugs] //, filters: [includePackage('io.jenkins.plugins.analysis.*')]
    }

    recordIssues(
            enabledForFailure: true, aggregatingResults: true,
            tools: [java(healthy: 10, unhealthy: 100, minimumSeverity: 'HIGH'), checkStyle(pattern: 'checkstyle-result.xml', reportEncoding: 'UTF-8')]
    )

    stage('SonarQube analysis') {
        def sonarqubeScannerHome = tool name: 'SonarQube Scanner'

        withSonarQubeEnv('SonarQube') {
            sh "${sonarqubeScannerHome}/bin/sonar-scanner -Dproject.settings=./sonar-project.properties"
        }

    }

    // No need to occupy a node
    stage("SonarQube Quality Gate"){
        timeout(time: 1, unit: 'MINUTES') { // Just in case something goes wrong, pipeline will be killed after a timeout
            def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
            if (qg.status != 'OK') {
                error "Pipeline aborted due to quality gate failure: ${qg.status}"
            }
        }
    }

    stage('Build') {
        echo "3. Install"
        sh "mvn clean install; cd $project_module ; mvn clean install"
    }

}






