WITH SNYK AND HEROKU

version: 2.1
orbs:
  node: circleci/node@5.0.1
  docker: circleci/docker@2.1.4 #docker orb
  heroku: circleci/heroku@2.0.0
  snyk: snyk/snyk@1.5.0 #3rd party orb of snyk

jobs:
  build: #build, set up 'dependencies'
    docker: #container to handle the steps
      - image: cimg/node:16.10
    steps:
      - checkout #copies the current directory of the project
      - node/install-packages:
          pkg-manager: npm
      - run: |
          echo "Installing dependencies..."
          npm install
  test:
    docker:
      - image: cimg/node:16.10
    steps:
      - checkout
      - node/install-packages:
          pkg-manager: npm
      - run: |
          echo "Running tests..."
          npm run test
  # configure a job called build-and-push to publish docker image to the docker hub registry only when the test job passed.
  build-and-push:
    executor: docker/docker #State the environment that we're use
    steps:
      - setup_remote_docker #This allows use to run docker commands just in case
      - checkout
      - docker/check
      - docker/build:
          image: clumped/education-space
          tag: << pipieline.git.tag >>
      - docker/push:
          image: clumped/education-space
          tag: << pipieline.git.tag >>
  scan: #scan for security issues job
        docker:
            - image: cimg/node:16.10
        environment: #can set values to use throughout the job
            IMAGE_NAME: clumped/education-space
        steps:
            - checkout
            - setup_remote_docker      
            - docker/check
            - run: docker build -t $IMAGE_NAME .
            - snyk/scan: 
                docker-image-name: $IMAGE_NAME
                severity-threshold: high #flags only high/severe 
  # deploy:
  #   docker: 
  #     - image: cimg/node16.10
  #   steps:
  #     - setup_remote_docker
  #     - heroku/install
  #     - checkout
  #     - run:
  #       name: Heroku Container Push #name of gorup of commands you are running
  #       command: 
  #         heroku container:login
  #         heroku container:push web -a appname #push > pushes image to heroku's container registry
  #         heroku container:release web -a appname #release deploys the image from heroku's container registry
  
workflows:
  simple_workflow:
    jobs:
      - build:
          filters:
            branches:
              only:
                - main
      - test:
          requires:
            - build
          filters:
            branches:
              only:
                - main
      - build-and-push:        #should run if a tag is pushed, but NOT if a branch is pushed
          requires:
            - test
          filters:
            tags:
              only: /^v.*/      #only considers tags that start with 'v'
            branches:
              ignore: /.*/    #ignores ALL branches
      # - deploy:
      #     requires:
      #       - build-and-push
      #     filters:
      #       tags:
      #         only:
      #           - /v[0-9]+\.[0-9]+\.[0-9]+/
      - scan:
          requires:
            - build
          filters:
            branches:
              only: main