// INFO 540 Exercise 8 - Jack Liu / 606406
// Secrets are supplied only by Jenkins Credentials, never by this repository.
node {
  withEnv([
    'JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64',
    'PATH+BUILD_JDK=/usr/lib/jvm/java-11-openjdk-amd64/bin',
    'AZURE_SUBSCRIPTION_ID=9b792f8f-b1ff-4537-8d83-fe898ace7fc4',
    'AZURE_TENANT_ID=760422ab-cf0f-45e7-82ec-23c3a2541163',
    'AZURE_RESOURCE_GROUP=jenkins-get-started-rg',
    'AZURE_WEBAPP_NAME=info540-jl606406-ex8',
    "AZURE_CONFIG_DIR=${pwd(tmp: true)}/azure-cli-${env.BUILD_NUMBER}"
  ]) {
    stage('Checkout') {
      checkout scm
    }
    stage('Maven WAR build') {
      sh '''
        java -version
        mvn --batch-mode --no-transfer-progress clean package
        test -s target/calculator-1.0.war
        ls -lh target/calculator-1.0.war
        sha256sum target/calculator-1.0.war
      '''
      archiveArtifacts artifacts: 'target/calculator-1.0.war', fingerprint: true
    }
    def hostName = ''
    withCredentials([usernamePassword(credentialsId: 'AzureServicePrincipal',
      usernameVariable: 'AZURE_CLIENT_ID', passwordVariable: 'AZURE_CLIENT_SECRET')]) {
      try {
        stage('Azure identity and subscription') {
          sh '''
            set +x
            umask 077
            mkdir -p "$AZURE_CONFIG_DIR"
            az login --service-principal --username "$AZURE_CLIENT_ID" --password "$AZURE_CLIENT_SECRET" --tenant "$AZURE_TENANT_ID" --only-show-errors --output none
            az account set --subscription "$AZURE_SUBSCRIPTION_ID"
            test "$(az account show --query id --output tsv)" = "$AZURE_SUBSCRIPTION_ID"
            az account show --query '{name:name,id:id,tenantId:tenantId}' --output json
            az group show --subscription "$AZURE_SUBSCRIPTION_ID" --name "$AZURE_RESOURCE_GROUP" --query '{name:name,id:id}' --output json
          '''
        }
        stage('Deploy WAR to App Service') {
          sh '''
            set +x
            test "$(az account show --query id --output tsv)" = "$AZURE_SUBSCRIPTION_ID"
            az webapp deploy --subscription "$AZURE_SUBSCRIPTION_ID" --resource-group "$AZURE_RESOURCE_GROUP" --name "$AZURE_WEBAPP_NAME" --src-path target/calculator-1.0.war --type war --async false --timeout 600000 --only-show-errors --output json
          '''
          hostName = sh(returnStdout: true, script: '''
            set +x
            test "$(az account show --query id --output tsv)" = "$AZURE_SUBSCRIPTION_ID"
            az webapp show --subscription "$AZURE_SUBSCRIPTION_ID" --resource-group "$AZURE_RESOURCE_GROUP" --name "$AZURE_WEBAPP_NAME" --query defaultHostName --output tsv
          ''').trim()
        }
      } finally {
        sh '''
          set +x
          az logout --output none 2>/dev/null || true
          rm -rf "$AZURE_CONFIG_DIR"
        '''
      }
    }
    stage('Verify deployed endpoints') {
      withEnv(["APP_URL=https://${hostName}"]) {
        sh '''
          python3 - <<'PY'
import json, os, time, urllib.request
base = os.environ['APP_URL']
paths = ['/api/calculator/ping', '/api/calculator/add?x=23&y=33']
for attempt in range(18):
    try:
        replies = []
        for path in paths:
            with urllib.request.urlopen(base + path, timeout=25) as response:
                replies.append(response.read().decode())
        assert 'Welcome to Java Web App' in replies[0], 'Unexpected ping content'
        assert json.loads(replies[1])['result'] == 56, 'Unexpected sum'
        for path, reply in zip(paths, replies):
            print(base + path)
            print(reply)
        print('Both deployed endpoints verified; result=56.')
        break
    except Exception as error:
        if attempt == 17:
            raise
        print('Waiting for initial App Service startup:', type(error).__name__)
        time.sleep(10)
PY
        '''
      }
    }
  }
}
