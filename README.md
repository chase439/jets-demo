# Jets Project

I used AWS Cloud9 for development and got this project working on 5/6/2021. Deployment was successfully. API endpoints work.

My procedure includes:

  - create an admin role or jets minimal deploy role: https://rubyonjets.com/docs/extras/minimal-deploy-iam/cli/
  - create Cloud9 env
  - config IAM role of Cloud9
    - Stop/edit/start EC2 of cloud9 to use admin IAM role
    - Disable cloud9 managed temporary credentials from its Preferences to use EC2 IAM Role
  - Install and use ruby 2.7 (supported by lambda layers)
  - install latest bundler b/c it was using version 1.
    - verify you have no ~/.bundle/config
  - install jq: sudo yum install jq
  - Create RDS: latest MariaDB, jets-demo, admin password, left everything else default
    - admin, D********
    - test connection: `mysql -h $HOST -P 3306  -u admin -p`
    - ssl ca = rds-ca-2019; VPC same as EC2
    - Allow remote connection to RDS
      - https://www.youtube.com/watch?v=83fbh9MFWos
      - changed RDS inbound security rule to allow 0.0.0.0/0 for MYSQL
        - `MYSQL/Aurora` `TCP` `3306` `0.0.0.0/0`
    - PASSWORD=&#96;openssl rand -base64 8&#96;
    - `APPLICATION=demo`
    - `HOST=xxxx`
    - `mysql -h $HOST -u admin --execute="CREATE USER '${APPLICATION}'@'%' IDENTIFIED BY '${PASSWORD}'" -p`
      - `SHOW GRANTS FOR 'admin'@'%';`
      - You can't grant SUPER privilege on RDS, so you can't use the ALL PRIVILEGES shortcut.This is an RDS restriction.
    - `mysql -h $HOST -u admin --execute="CREATE DATABASE ${APPLICATION}_development" -p`
    - `mysql -h $HOST -u admin --execute="USE ${APPLICATION}_development; GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER, CREATE TEMPORARY TABLES, CREATE VIEW, EVENT, TRIGGER, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, EXECUTE ON  ${APPLICATION}_development.* TO ' ${APPLICATION}'@'%';" -p`
    - `mysql -h $HOST -u admin --execute="CREATE DATABASE ${APPLICATION}_test" -p`
    - `mysql -h $HOST -u admin --execute="USE ${APPLICATION}_test; GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER, CREATE TEMPORARY TABLES, CREATE VIEW, EVENT, TRIGGER, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, EXECUTE ON demo_test.* TO 'demo'@'%';" -p`
  - update .env.development and .env.test to include database credentials
  - touch db/seeds.rb # for jets db:setup
  - Upgrade Node verison
    - Install NVM: `curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash`
    - `npm install yarn -g`
  - add .nvmrc and .ruby-version
  - `jets webpacker:install:erb`
  - `rm app/javascript/packs/hello_erb.js.erb`
  - bundle && yarn
  - `jets db:setup`
  - `jets serve --port $PORT` # https://docs.aws.amazon.com/cloud9/latest/user-guide/app-preview.html#app-preview-preview-app (8080, 8081, or 8082)
  - Cloud9 > Preview > Preview Running Application
  - Fix gem layer for deploy
    - `bundle config path ~/jets-gems/ruby/2.7.0`
    - `mkdir -p ~/jets-gems/ruby/2.7.0`
    - `bundle`
    - `cd ~/jets-gems`
    - `zip -r demo_gems ruby`  #=> demo_gems.zip
    - `aws s3 cp demo_gems.zip s3://jets-demo-rails` # upload zip to s3
    - Create lambda layer and point it to uploaded zip file in S3
    - edit config/application.rb, add gem lambda layer
  - `jets deploy`
  - `jets url` # provides url of the lambda endpoint
  - View Log (Cloudwatch) to troubleshoot any app errors. There's a log group for each lambda function.
  - Config Custom Domain Name
