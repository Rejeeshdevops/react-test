pipeline {
  agent any
  // If you want to force a specific agent/label, use:
  // agent { label 'node21' }

  environment {
    // Pull secrets from Jenkins Credentials (IDs must match your saved secrets)
    AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
    AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
    AWS_REGION            = credentials('AWS_REGION')
    S3_BUCKET             = credentials('S3_BUCKET')
  }

  options {
    timestamps()
  }

  stages {
    stage('Node setup') {
      steps {
        sh '''
          set -e
          # Use Node 18 via nvm if you installed nvm for the jenkins user
          export NVM_DIR="$HOME/.nvm"
          [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
          if command -v nvm >/dev/null 2>&1; then
            echo "Using nvm to set Node 18..."
            nvm install 18
            nvm use 18
          else
            echo "nvm not found; using system Node"
          fi

          node -v
          npm -v
          aws --version || true
        '''
      }
    }

    stage('Install & Build') {
      steps {
        sh '''
          set -e
          echo "Installing deps..."
          npm ci || npm install

          echo "Building..."
          npm run build

          echo "Verifying build output..."
          ls -al ./dist
        '''
      }
    }

    stage('Deploy to S3') {
      steps {
        sh '''
          set -e
          if [ -z "$AWS_REGION" ] || [ -z "$S3_BUCKET" ]; then
            echo "AWS_REGION or S3_BUCKET is empty. Check Jenkins Credentials IDs."
            exit 1
          fi

          echo "Testing AWS identity..."
          aws sts get-caller-identity

          echo "Syncing dist -> s3://$S3_BUCKET (region: $AWS_REGION)"
          aws s3 sync ./dist s3://$S3_BUCKET \
            --region "$AWS_REGION" \
            --exact-timestamps \
            --delete \
            --cache-control "no-cache, no-store, must-revalidate"
        '''
      }
    }

    // OPTIONAL: uncomment if you have a CloudFront distribution
    // and add a credential/secret text with ID CLOUDFRONT_DISTRIBUTION_ID
    /*
    stage('Invalidate CloudFront') {
      environment {
        CLOUDFRONT_DISTRIBUTION_ID = credentials('CLOUDFRONT_DISTRIBUTION_ID')
      }
      steps {
        sh '''
          set -e
          if [ -z "$CLOUDFRONT_DISTRIBUTION_ID" ]; then
            echo "CLOUDFRONT_DISTRIBUTION_ID is empty—skipping invalidation."
            exit 0
          fi
          echo "Invalidating CloudFront..."
          aws cloudfront create-invalidation \
            --distribution-id "$CLOUDFRONT_DISTRIBUTION_ID" \
            --paths "/*" \
            --region "$AWS_REGION"
        '''
      }
    }
    */
  }
}
