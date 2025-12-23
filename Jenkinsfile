pipeline {
  agent any

  stages {

    /* ---------------- INSTALL ---------------- */
    stage('Install Checkov') {
      steps {
        sh '''
          set +x
          echo "▶ Ensuring Checkov is available"

          PYTHON_VERSION=$(python3 -c "import sys; print(f'{sys.version_info.major}.{sys.version_info.minor}')")
          USER_BIN="$HOME/Library/Python/${PYTHON_VERSION}/bin"
          export PATH="$PATH:$USER_BIN"

          if ! command -v checkov >/dev/null 2>&1; then
            pip3 install --user checkov --quiet
          fi

          echo "✔ Checkov ready: $(checkov --version)"
        '''
      }
    }

    /* ---------------- FULL SCAN ---------------- */
stage('Run Checkov Terraform Scan') {
  steps {
    sh '''
      set +x
      echo "▶ Running Checkov Terraform scan"

      PYTHON_VERSION=$(python3 -c "import sys; print(f'{sys.version_info.major}.{sys.version_info.minor}')")
      USER_BIN="$HOME/Library/Python/${PYTHON_VERSION}/bin"
      export PATH="$PATH:$USER_BIN"

      # -------- CLI output (for summary + Jenkins logs) --------
      checkov \
        --directory . \
        --framework terraform \
        --compact \
        --summary-position top \
        --output cli \
        | tee checkov.txt || true

      # -------- JSON output (for PR comment) --------
      checkov \
        --directory . \
        --framework terraform \
        --output json \
        > checkov.json || true
    '''
  }
}


    /* ---------------- PR DECORATION ---------------- */
    
        stage('Post or Update PR Comment') {
            when {
                expression { env.CHANGE_ID != null }
            }
            steps {
                withCredentials([
                    string(credentialsId: 'github-pat', variable: 'GITHUB_TOKEN')
                ]) {
                    sh '''
set -e

OWNER=$(echo "$GIT_URL" | sed -E 's#.*/([^/]+)/([^/.]+)(\\.git)?#\\1#')
REPO=$(echo "$GIT_URL" | sed -E 's#.*/([^/]+)/([^/.]+)(\\.git)?#\\2#')
PR=$CHANGE_ID

SUMMARY=$(grep -E "Passed checks:|Failed checks:|Skipped checks:" checkov.txt || true)

COMMENT=$(cat <<EOF
### 🔍 Checkov Terraform Scan Results

Repository: $REPO
PR: #$PR
Commit: $GIT_COMMIT

Summary:
$SUMMARY

❗ This PR introduces Terraform security findings.
Please review the Jenkins build logs for full Checkov details.

EOF
)

EXISTING_COMMENT_ID=$(curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/$OWNER/$REPO/issues/$PR/comments \
  | jq -r '.[] | select(.body | contains("### 🔍 Checkov Terraform Scan Results")) | .id' | head -n1)

if [ -n "$EXISTING_COMMENT_ID" ]; then
    echo "Updating existing PR comment ID: $EXISTING_COMMENT_ID"
    curl -s -X PATCH \
      -H "Authorization: Bearer $GITHUB_TOKEN" \
      -H "Accept: application/vnd.github+json" \
      https://api.github.com/repos/$OWNER/$REPO/issues/comments/$EXISTING_COMMENT_ID \
      -d "$(jq -n --arg body "$COMMENT" '{body: $body}')"
else
    echo "Creating new PR comment"
    curl -s -X POST \
      -H "Authorization: Bearer $GITHUB_TOKEN" \
      -H "Accept: application/vnd.github+json" \
      https://api.github.com/repos/$OWNER/$REPO/issues/$PR/comments \
      -d "$(jq -n --arg body "$COMMENT" '{body: $body}')"
fi
'''
                }
            }
        }
    }


  post {
    success {
      echo "✅ Checkov completed successfully"
    }
  }
}
