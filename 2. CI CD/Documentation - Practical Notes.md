In order to proceed, I would first have to develop a simple web application containing tests and endpoints in order to test out the CI/CD pipeline. I've built my application using Spring Boot with Java, since that is the one thing I know for sure how to do.

Because I don't have to build the backend too much, no authentication, no filter configs I've gone straight to the **Controller Class**:
```
package com.dianden0.controllers;  
  
import org.springframework.http.HttpStatus;  
import org.springframework.http.ResponseEntity;  
import org.springframework.web.bind.annotation.*;  
  
import java.time.LocalDateTime;  
import java.util.ArrayList;  
import java.util.Arrays;  
  
@RestController  
public class WebController {  
  
    private int numberOfVisits = 0;  
    public static ArrayList<String> tasks = new ArrayList<>(Arrays.asList("Task1", "Task2", "Task3", "Task4", "Task5"));  
  
    @RequestMapping("/")  
    public String home() {  
        numberOfVisits++;  
        return "Hello World!";  
    }  
  
    @GetMapping("/getTime")  
    public    String getTime(){  
        numberOfVisits++;  
        return LocalDateTime.now().toString();  
    }  
  
    @GetMapping("/getTasks")  
    public  String getTasks(){  
        numberOfVisits++;  
        return tasks.toString();  
    }  
  
    @PostMapping("/createTask")  
    public ResponseEntity<String> createTask(@RequestParam String task){  
        numberOfVisits++;  
        if (tasks.contains(task)){  
            return new ResponseEntity<>("Task Already Exists", HttpStatus.CONFLICT);  
        } else {  
            tasks.add(task);  
            return new ResponseEntity<>("Task Created Successfully", HttpStatus.CREATED);  
        }  
    }}
```

These are basic endpoints that could ensure the application works as intended when I'm gonna use Prometheus and Grafana.

Now as for the tests, in order to test one of the stages of the CI pipeline where we will check if the unit tests will run correctly, we got to first create them:

```
package com.dianden0;  
  
import com.dianden0.controllers.WebController;  
import org.junit.jupiter.api.BeforeEach;  
import org.junit.jupiter.api.Test;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;  
import org.springframework.test.web.servlet.MockMvc;  
  
// These static imports are required for the MockMvc syntax  
import java.util.ArrayList;  
import java.util.Arrays;  
  
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;  
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;  
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;  
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;  
import static org.junit.jupiter.api.Assertions.assertTrue;  
  
@WebMvcTest(WebController.class)  
public class UnitTests {  
  
    @Autowired  
    private MockMvc mockMvc;  
  
    @BeforeEach  
    void resetState() {  
  
        WebController.tasks = new ArrayList<>(Arrays.asList("Task1", "Task2", "Task3", "Task4", "Task5"));  
    }  
  
    @Test  
    void homeTest() throws Exception {  
        mockMvc.perform(get("/"))  
                .andExpect(status().isOk())  
                .andExpect(content().string("Hello World!"));  
    }  
  
    @Test  
    void getTasksTest() throws Exception {  
        mockMvc.perform(get("/getTasks"))  
                .andExpect(status().isOk())  
                .andExpect(content().string("[Task1, Task2, Task3, Task4, Task5]"));  
    }  
  
    @Test  
    void createTaskTest_Success() throws Exception {  
        mockMvc.perform(post("/createTask")  
                        .param("task", "New Pipeline Task"))  
                .andExpect(status().isCreated());  
  
        assertTrue(WebController.tasks.contains("New Pipeline Task"));  
    }  
  
    @Test  
    void createTaskTest_Conflict() throws Exception {  
        mockMvc.perform(post("/createTask")  
                        .param("task", "Task1"))  
                .andExpect(status().isConflict());  
    }  
}
```

Here we have basic unit tests for every endpoint.

I've tweaked a few more things, but we'll get to them later.

As per the pipeline itself, I'm using Gitlab CI/CD for building my pipeline. It's easy to use, has a lot more freedom than Github Actions and does not make me pay for its intended use.

To start, I've had to write the pipeline itself, we'll go over the entire thing in detail. In order to create the pipeline, we start with writing the `.gitlab-ci.yml`:

```
stages:
  - validate 
  - test       
  - compile
  - build
  - release
  - deploy

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_PIPELINE_ID

lint-job:
  image: eclipse-temurin:17-jdk-alpine
  stage: validate
  script:
    - echo "Starting lint validation..."
    - chmod +x ./mvnw
    - ./mvnw checkstyle:checkstyle
  artifacts:
    when: always
    reports:
      codequality:
        - target/checkstyle-result*.xml
  only:
    - branches


unit-test-job:
  image: eclipse-temurin:17-jdk-alpine
  stage: test  
  script:
    - chmod +x ./mvnw
    - ./mvnw clean test
  artifacts:
    when: always
    reports:
      junit:
        - target/surefire-reports/TEST-*.xml
  only:
    - branches

compile-maven-job:
  image: eclipse-temurin:17-jdk-alpine
  stage: compile
  script:
    - echo "compiling Maven project"
    - chmod +x ./mvnw
    - ./mvnw clean package
  artifacts:
    name: maven_build
    paths:
      - target/*.jar
    expire_in: 1 hour
  only:
    - branches

publish_gitlab_container_registry:
  stage: build
  needs:
    - compile-maven-job
    - unit-test-job
  image: docker:24.0.5
  services:
    - docker:24.0.5-dind
  script:
    - echo "Logging in to GitLab Registry..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin
    - echo "Building image..."
    - docker build -t $IMAGE_TAG .
    - echo "Pushing image to Registry..."
    - docker images
    - docker push $IMAGE_TAG
  only:
    - branches


semantic-versioning:
  image: node:lts
  stage: release
  needs:
    - job: publish_gitlab_container_registry
  variables:
    GITLAB_TOKEN: $GITLAB_SEMANTIC_VERSIONING_TOKEN
  script:
    - npx -y
      -p @semantic-release/commit-analyzer@9.0.2
      -p @semantic-release/git@10.0.1
      -p @semantic-release/gitlab@9.5.0
      -p @semantic-release/release-notes-generator@10.0.3
      -p @semantic-release/exec@6.0.3
      -p @semantic-release/changelog@6.0.2
      -p conventional-changelog-conventionalcommits@5.0.0
      -p semantic-release@19.0.5
      semantic-release
  only:
    - master

build-image-latest:
  stage: release
  needs:
    - compile-maven-job
    - semantic-versioning
  image: docker:24.0.5
  services:
    - docker:24.0.5-dind
  script:
    - echo "Logging in to GitLab Registry..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin
    - echo "Building latest image..."
    - docker build -t $CI_REGISTRY_IMAGE:latest .
    - echo "Pushing image to Registry..."
    - docker images
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - master


update-manifest-job:
  stage: deploy
  needs:
    - build-image-latest
    - publish_gitlab_container_registry
  image:
    name: alpine/git
    entrypoint: [""]

  script:
    - echo "Updating Kubernetes manifests with new image tag..."
    - git config --global user.email "pipeline@gitlab.com"
    - git config --global user.name "GitLab CI"

    - git clone https://oauth2:${GITLAB_SEMANTIC_VERSIONING_TOKEN}@gitlab.com/dianden0-group/simple_web_app.git
    - cd simple_web_app

    - 'sed -i "s|image: registry.gitlab.com/dianden0-group/simple_web_app:.*|image: registry.gitlab.com/dianden0-group/simple_web_app:${CI_PIPELINE_ID}|g" manifests/web-app-deployment.yaml'
    - git add manifests/web-app-deployment.yaml
    - 'git commit -m "chore: update image tag to ${CI_PIPELINE_ID} [skip ci]"'
    -  git push origin master
  only:
    - master



deploy-job:
  stage: deploy
  needs:
    - job: update-manifest-job
    - job: semantic-versioning
    - job: build-image-latest
  environment: production
  script:
    - echo "Deploying application..."
    - echo "Application successfully deployed."
  only:
    - master

```

The pipeline consists of  6 stages:
  - validate 
  - test       
  - compile
  - build
  - release
  - deploy

Each stage corresponds to a different pipeline stage and each does a completely different thing

### validate
The `validate` stage contains our `lint-job`, the one in charge of checking the code for syntax issues.  I'm using an **eclipse** image with **alpine** installed with Java JDK 17 and the linter in this case is a Java linter called **Checkstyle**. Currently, it only runs and its logs are being put into an artifact. 

### test
The test stage is where our unit tests come into play. By using the same **eclipse** image from earlier, we use the maven wrapper of our projects to run the tests. This report is also being turned into an artifact and thus can be accessed through the registry. If the build is unsuccessful, and some of the tests show the application does not function as intended the pipeline would stop. 

### compile
The compile stage is where the project is being compiled. Using the same image from earlier we run and make sure the project is compiled correctly, this is to ensure we are able to build an **image** of our app without any problems

### build
The build portion uses a **docker** image to build the image from our project. For this to work we created a **Dockerfile** inside of our project's repository which gives instructions to docker for how to build the image. We also use the Docker CLI to login into our Gitlab's registry. This is needed for us to be able to push the newly made image into our container registry, where the image will be saved with the corresponding tag attributed to it.

### release
This is the stage where we create a new release for our application. Using a **node**.js image we with the help of a few GitHub repositories called semanti-release create a dynamic release that calculates the appropriate version for our new release based on our commit's message. A major update will warrant an increase like this: `x+1.y.z`  and so forth. 

### deploy
The stage by itself does nothing to "deploy" the image, however it has a job for updating our `manifest` files for ArgoCD, which will come in handy later. 



## The CD Part

Since we want our image to be part of our **Kubernetes** cluster, we need to automate the deployment to it. 

Firstly, I've configured a GitLab agent which sits in my Kubernetes cluster and maintains connection with my GitLab repository. However, using it would prove to be a liability in my case. Since I wish to automate the deployment of the application, using the CLI of the agent to do so will be a tedious and an unoptimized way of handling the automation. For this reason the deployed agent will probably be removed in the future for its lack of uses. 

```
kubectl create secret docker-registry gitlab-registry-secret \ --docker-server=registry.gitlab.com \ --docker-username=<YOUR_USERNAME> \ --docker-password=<YOUR_PASSWORD> \ --docker-email=<YOUR_EMAIL> \ -n my-app-env

kubectl get secret gitlab-registry-secret -o yam

helm repo add gitlab https://charts.gitlab.io
helm repo update
helm upgrade --install k3s-example-agent gitlab/gitlab-agent \
    --namespace gitlab-agent-k3s-example-agent \
    --create-namespace \
    --set config.token=YOURTOKEN \
    --set config.kasAddress=wss://kas.gitlab.com

```

Instead, I went ahead and configured **ArgoCD**, a GitOps CD tool for Kubernetes to deploy the image automatically once a commit is being made. By following this [guide](https://argo-cd.readthedocs.io/en/stable/#quick-start) and this [one](https://argo-cd.readthedocs.io/en/stable/getting_started/) I've set up the argo cd server side and client into my cluster, and from there I've used the built-in UI overlay accessed through the browser on my Ubuntu machine to configure the deployment. In the end, it looks like this:


<img width="1414" height="372" alt="image" src="https://github.com/user-attachments/assets/6f1d1bc1-a701-4e30-b306-b0b8f55be776" />

These are a few commands I've used to configure the ArgoCD further:
```
kubectl create namespace argocd kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}
```


These are links to a few more sites I've used during the development:
https://checkstyle.org/writingchecks.html
https://www.geeksforgeeks.org/devops/what-is-ci-cd/
