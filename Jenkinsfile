#!groovy

library identifier: '3scale-toolbox-jenkins@master',
        retriever: modernSCM([$class: 'GitSCMSource',
                              remote: 'https://github.com/rh-integration/3scale-toolbox-jenkins.git',
                              traits: [[$class: 'jenkins.plugins.git.traits.BranchDiscoveryTrait']]])

def service = null

node() {
  stage('Checkout Source') {
    checkout scm
  }

  stage("Prepare") {
    service = toolbox.prepareThreescaleService(
        openapi: [filename: "city.yaml"],
        environment: [ baseSystemName: "city-service-thorntail",
                       privateBaseUrl: params.PRIVATE_BASE_URL,
                       stagingPublicBaseURL: "https://city-apicast-staging." + params.PUBLIC_STAGING_WILDCARD_DOMAIN,
                       productionPublicBaseURL: "https://city-apicast-production." + params.PUBLIC_PRODUCTION_WILDCARD_DOMAIN ],
        toolbox: [ openshiftProject: params.NAMESPACE,
                   destination: params.TARGET_INSTANCE,
                   image: "quay.io/redhat/3scale-toolbox:master", // TODO: remove me once the final image is released
                   insecure: params.DISABLE_TLS_VALIDATION == "yes",
                   secretName: params.SECRET_NAME],
        service: [:],
        applications: [
            [ name: "my-test-app", description: "This is used for tests", plan: "test", account: params.DEVELOPER_ACCOUNT_ID ]
        ],
        applicationPlans: [
          [ systemName: "test", name: "Test", defaultPlan: true, published: true ],
          [ systemName: "silver", name: "Silver" ],
          [ systemName: "gold", name: "Gold" ],
        ]
    )

    //echo "toolbox version = " + service.toolbox.getToolboxVersion()
  }

  stage("Import OpenAPI") {
    service.importOpenAPI()
    echo "Service with system_name ${service.environment.targetSystemName} created !"
  }

  stage("Create an Application Plan") {
    service.applyApplicationPlans()
  }

  stage("Create an Application") {
    service.applyApplication()
  }

  stage("Run integration tests") {
    // The staging and production base URLs are computed by the shared library
    def proxy = [
        endpoint: service.environment.productionPublicBaseURL,
        sandbox_endpoint: service.environment.stagingPublicBaseURL
    ]
    sh """set -e
    echo "Public Staging Base URL is ${proxy.sandbox_endpoint}"
      curl -sfk -w "List cities: %{http_code}\n" -o /dev/null ${proxy.sandbox_endpoint}/v1/countries/BE/cities
    """
  }

  stage("Promote to production") {
    service.promoteToProduction()
  }

}
