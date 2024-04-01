# Connecting GH Actions to Private AWS Database

### 1. Configure AWS resources (more details later)

a. Inside your VPC, create a private and public subnet

b. Put your database resource in the private subnet

c. Set an Internet Gateway to the public subnet route table and Nat Gateway to the private subnet route table

d. Set up security group for bastion EC2 instance

e. Set database security group allow bastion EC2 instance security group to access via port 3306

### 2. Set up bastion instance

a. Create an EC2 instance in the public subnet (this will be our bastion host)

b. Create IAM role with the policy for:

- AWS SSM Managed Instance Core
- SSM Start Session

      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Action": "ssm:StartSession",
            "Resource": [
              "arn:aws:ec2:(region):(account-id):instance/(instance-id)”,
                "arn:aws:ssm:*:*:document/AWS-StartSSHSession"
            ]
          }
        ]
      }

c. Set the IAM role trust relationship:

        {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Effect": "Allow",
              "Principal": {
                "Service": "ec2.amazonaws.com"
              },
              "Action": "sts:AssumeRole"
            },
            {
              "Effect": "Allow",
              "Principal": {
                "Service": "ssm.amazonaws.com"
              },
              "Action": "sts:AssumeRole"
            }
          ]
        }

d. Attach the IAM role to the EC2 instance

### 3. Set up IAM role for GH Actions access (more details later)

a. Create OIDC identity provider for your github repository `AssumeRoleWithWebIdentity`

    statement {
      effect  = "Allow"
      actions = ["sts:AssumeRoleWithWebIdentity"]
      principals {
        type        = "Federated"
        identifiers = ["arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/token.actions.githubusercontent.com"]
      }
      condition {
        test     = "StringEquals"
        variable = "token.actions.githubusercontent.com:aud"
        values   = ["sts.amazonaws.com"]
      }
      condition {
        test     = "StringLike"
        variable = "token.actions.githubusercontent.com:sub"
        values   = ["repo:(your-github)/(repo-name):*"]
      }

}

b. Create SSM start session policy

    {
      Version : "2012-10-17",
      Statement : [
        {
          Sid : "SessionManagerStartSessionAccess",
          Effect : "Allow",
          Action : "ssm:StartSession",
          Resource : [
            "arn:aws:ec2:${var.region}:${data.aws_caller_identity.current.account_id}:instance/${aws_instance.bastion.id}",
            "arn:aws:ssm:*:*:document/AWS-StartSSHSession"
          ],
        },
      ],
    }

c. Create IAM role with `OIDC identity provider` as assume-role-policy and `SSM start session policy` as managed policy

d. Add other necessary policy to the role (ex: secrets manager, etc)

- [optional] Add policy for accessing secrets manager to store your secrets (instance key, database secrets)

      {
        Version : "2012-10-17",
        Statement : [
          {
            Sid : "SecretManagerGetValue",
            Effect : "Allow",
            Action : "secretsmanager:GetSecretValue",
            Resource : [
              "arn:aws:secretsmanager:${var.region}:${data.aws_caller_identity.current.account_id}:secret:(instance-key)*",
              "arn:aws:secretsmanager:${var.region}:${data.aws_caller_identity.current.account_id}:secret:(your-database-secret)*",
            ]
          }
        ]
      }

### 4. GH Actions Workflow

a. Assume the OIDC role using `aws-actions/configure-aws-credentials@v3`

    - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v3
        with:
          aws-region: (AWS-region)
          role-to-assume: "arn:aws:iam::(AWS-id):role/(OIDC-role)"
          role-session-name: (session-name)

b. Configure SSH config for SSM

    - name: Configure SSH config for SSM
        run: |
          mkdir -p ~/.ssh
          echo 'host i-* mi-*' >> ~/.ssh/config
          echo '    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters '\''portNumber=%p'\''"' >> ~/.ssh/config

c. [optional] Get bastion instance private key from Secrets Manager

      - name: Get bastion host secret
        run: |
          aws secretsmanager get-secret-value --secret-id (bastion-key).pem | jq -r '.SecretString' | jq -r '.["(bastion-key).pem"]' | sed 's/BEGIN RSA PRIVATE KEY-----/&\n/' | sed 's/-----END RSA PRIVATE KEY-----/\n&/' | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//' > (bastion-key).pem

d. Set private key permission

      - name: Set key permission
        run: |
          chmod 400 bastion-key.pem

e. [optional] Get database secrets and store as GH output

      (variable depends of your secrets manager format)
      - name: Get database secrets
        id: get_db_secrets
        run: |
          aws secretsmanager get-secret-value --secret-id ${{ env.DB_SECRET_ID }} | jq --raw-output '.SecretString' > creds.json
          echo "USERNAME=$(jq -r .username creds.json)" >> $GITHUB_OUTPUT
          echo "PASSWORD=$(jq -r .password creds.json)" >> $GITHUB_OUTPUT
          echo "DB_INSTANCE_ID=$(jq -r .dbInstanceIdentifier creds.json)" >> $GITHUB_OUTPUT
          echo "ENDPOINT=$(jq -r .host creds.json)" >> $GITHUB_OUTPUT
          echo "ENGINE=$(jq -r .engine creds.json)" >> $GITHUB_OUTPUT
          echo "PORT=$(jq -r .port creds.json)" >> $GITHUB_OUTPUT
          rm creds.json

f. Connect to the instance using SSH via SSM

      - name: Flyway migrate via bastion host
        run: |
          ssh -o "StrictHostKeyChecking no" -i "(bastion-key).pem" ec2-user@(bastion-id) echo "Hello World!"

g. [optional] Migrate using Flyway

- Copy migration scripts to instance

      - name: Copy SQL migration scripts to bastion host
        run: |
          ssh -o "StrictHostKeyChecking no" -i "(bastion-key).pem" ec2-user@(bastion-id) \
          'mkdir -p /home/ec2-user/flyway/sql'
          scp -o "StrictHostKeyChecking no" -i "(bastion-key).pem" -r (your-script-path) ec2-user@(bastion-id):/home/ec2-user/flyway/sql

- Run Flyway repair in case of previous migration failure

      - name: Flyway repair via bastion host
        run: |
          ssh -o "StrictHostKeyChecking no" -i "(bastion-key).pem" \
          -L 3306:${{ steps.get_db_secrets.outputs.ENDPOINT }}:3306 \
          ec2-user@(bastion-id) \
          flyway -url=jdbc:mysql://${{ steps.get_db_secrets.outputs.ENDPOINT }}:${{ steps.get_db_secrets.outputs.PORT }}/(database-name) \
          -user=${{ steps.get_db_secrets.outputs.USERNAME }} -password=${{ steps.get_db_secrets.outputs.PASSWORD }} -schemas=(database-name) \
          repair

- Flyway migrate

      - name: Flyway migrate via bastion host
        run: |
          ssh -o "StrictHostKeyChecking no" -i "(bastion-key).pem" \
          -L 3306:${{ steps.get_db_secrets.outputs.ENDPOINT }}:3306 \
          ec2-user@(bastion-id) \
          flyway -url=jdbc:mysql://${{ steps.get_db_secrets.outputs.ENDPOINT }}:${{ steps.get_db_secrets.outputs.PORT }}/(database-name) \
          -locations=filesystem:flyway/sql -user=${{ steps.get_db_secrets.outputs.USERNAME }} -password=${{ steps.get_db_secrets.outputs.PASSWORD }} -schemas=(database-name) \
          -createSchemas=false migrate

h. [optional] Clean database

      - name: Flyway clean database
        run: |
          ssh -o "StrictHostKeyChecking no" -i "(bastion-key).pem" \
          -L 3306:${{ steps.get_db_secrets.outputs.ENDPOINT }}:3306 \
          ec2-user@(bastion-id) \
          flyway -url=jdbc:mysql://${{ steps.get_db_secrets.outputs.ENDPOINT }}:${{ steps.get_db_secrets.outputs.PORT }}/(databa) \
          -user=${{ steps.get_db_secrets.outputs.USERNAME }} -password=${{ steps.get_db_secrets.outputs.PASSWORD }} -schemas=(databa) \
          clean -outputType=json
          ssh -o "StrictHostKeyChecking no" -i "(bastion-key).pem" \
          -L 3306:${{ steps.get_db_secrets.outputs.ENDPOINT }}:3306 \
          ec2-user@(bastion-id) \
          flyway -url=jdbc:mysql://${{ steps.get_db_secrets.outputs.ENDPOINT }}:${{ steps.get_db_secrets.outputs.PORT }}/(databa) \
          -user=${{ steps.get_db_secrets.outputs.USERNAME }} -password=${{ steps.get_db_secrets.outputs.PASSWORD }} -schemas=(databa) \
          repair
          ssh -o "StrictHostKeyChecking no" -i "(bastion-key).pem" \
          -L 3306:${{ steps.get_db_secrets.outputs.ENDPOINT }}:3306 \
          ec2-user@(bastion-id) \
          rm -rf /home/ec2-user/flyway/sql/*
