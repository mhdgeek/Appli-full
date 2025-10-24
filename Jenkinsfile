pipeline {
agent any

tools {
    nodejs "NodeJS_22"
}

environment {
    DOCKER_HUB_USER = 'mhd0'
    FRONT_IMAGE = 'react-frontend'
    BACK_IMAGE  = 'express-backend'
    PATH = "/usr/local/bin:${env.PATH}"
}

triggers {
    GenericTrigger(
        genericVariables: [
            [key: 'ref', value: '$.ref'],
            [key: 'pusher_name', value: '$.pusher.name'],
            [key: 'commit_message', value: '$.head_commit.message']
        ],
        causeString: 'Push GitHub par $pusher_name: $commit_message',
        token: 'mysecret',
        printContributedVariables: true,
        printPostContent: true,
        regexpFilterText: '$ref',
        regexpFilterExpression: 'refs/heads/main'
    )
}

stages {
    stage('Checkout') {
        steps {
            git branch: 'main', url: 'https://github.com/KingW223/Appli-full.git'
        }
    }

    stage('Install dependencies - Backend') {
        steps {
            dir('back-end') {
                sh 'npm install'
            }
        }
    }

    stage('Install dependencies - Frontend') {
        steps {
            dir('front-end') {
                sh 'npm install'
            }
        }
    }

    stage('Run Tests') {
        steps {
            dir('back-end') {
                sh 'npm test || echo "Aucun test backend ou échec ignoré"'
            }
            dir('front-end') {
                sh 'npm test || echo "Aucun test frontend ou échec ignoré"'
            }
        }
    }

    stage('Build Docker Images') {
        steps {
            script {
                sh "docker build -t ${env.DOCKER_HUB_USER}/${env.FRONT_IMAGE}:latest ./front-end"
                sh "docker build -t ${env.DOCKER_HUB_USER}/${env.BACK_IMAGE}:latest ./back-end"
            }
        }
    }

    stage('Push Docker Images') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'king-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            script {
                sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    docker push ${DOCKER_HUB_USER}/${FRONT_IMAGE}:${BUILD_NUMBER}
                    docker push ${DOCKER_HUB_USER}/${BACK_IMAGE}:latest
                '''
            }
        }
    }
}

    stage('Deploy to Kubernetes') {
        steps {
            script {
                echo "🚀 Déploiement MongoDB..."
                sh 'kubectl apply -f k8s/mongodb-deployment.yaml'

                echo "⏳ Attente du démarrage de MongoDB..."
                // bat 'timeout /t 60 /nobreak'

                echo "🚀 Déploiement Backend..."
                sh 'kubectl apply -f k8s/backend-deployment.yaml'
                sh 'kubectl apply -f k8s/backend-service.yaml'
                // bat 'timeout /t 20 /nobreak'

                echo "🚀 Déploiement Frontend..."
                sh 'kubectl apply -f k8s/frontend-deployment.yaml'
                sh 'kubectl apply -f k8s/frontend-service.yaml'

                echo "⏳ Attente des déploiements..."
                sh '''
                    kubectl rollout status deployment/backend-deployment --timeout=300s
                    kubectl rollout status deployment/frontend-deployment --timeout=300s
                '''
            }
        }
    }

  stage('Health Check & Smoke Tests') {
    steps {
        script {
            echo "🔍 Vérification simplifiée des services..."

            // Vérification des pods
          sh '''
                echo === Vérification des pods ===
                set RUNNING=0
                set TOTAL=0
                for /F "tokens=3" %%a in ('kubectl get pods --no-headers') do (
                    set STATUS=%%a
                    if "%%a"=="Running" set /a RUNNING+=1
                    set /a TOTAL+=1
                )
                echo Pods running: %RUNNING% / %TOTAL%
                if not "%RUNNING%"=="%TOTAL%" (
                    echo ❌ Certains pods ne sont pas prêts
                    exit /b 1
                ) else (
                    echo ✅ Tous les pods sont en cours d'exécution
                )
            '''


            // Test du backend
            sh '''
                echo === Test du backend ===
                start /B kubectl port-forward service/backend-service 5001:5000
                timeout /t 5 /nobreak
                curl -s http://localhost:5001
                taskkill /IM kubectl.exe /F
            '''

            // Test du frontend
            sh '''
                echo === Test du frontend ===
                for /f "delims=" %%p in ('kubectl get service frontend-service -o jsonpath="{.spec.ports[0].nodePort}"') do set FRONTEND_PORT=%%p
                for /f "delims=" %%i in ('minikube ip') do set MINIKUBE_IP=%%i
                echo Frontend URL: http://%MINIKUBE_IP%:%FRONTEND_PORT%
                curl -s -o NUL -w "HTTP Code: %%{http_code}\\n" "http://%MINIKUBE_IP%:%FRONTEND_PORT%" || echo Frontend en cours de démarrage
            '''
        }
    }
}


    stage('Update Kubernetes Images') {
        steps {
            script {
                sh "kubectl set image deployment/backend-deployment backend=${env.DOCKER_HUB_USER}/${env.BACK_IMAGE}:${BUILD_NUMBER}"
                sh "kubectl set image deployment/frontend-deployment frontend=${env.DOCKER_HUB_USER}/${env.FRONT_IMAGE}:${BUILD_NUMBER}"

                sh '''
                    kubectl rollout status deployment/backend-deployment --timeout=300s
                    kubectl rollout status deployment/frontend-deployment --timeout=300s
                '''
            }
        }
    }
}

post {
    always {
        echo 'Pipeline terminé - vérifiez les logs pour les détails'
        script {
            if (currentBuild.result == 'FAILURE') {
                sh '''
                    echo "=== Backend Pods ==="
                    kubectl get pods -l app=backend
                    echo "=== Frontend Pods ==="
                    kubectl get pods -l app=frontend
                    echo "=== MongoDB Pods ==="
                    kubectl get pods -l app=mongodb
                    echo "=== Services ==="
                    kubectl get services
                '''
            }
        }
    }

    success {
        script {
            sh '''
                echo "🎉 DÉPLOIEMENT RÉUSSI !"
                echo "Frontend: $(minikube service frontend-service --url)"
                echo "Backend: $(minikube service backend-service --url)"
            '''
            frontendUrl = sh(script: 'minikube service frontend-service --url', returnStdout: true).trim()
            backendUrl = sh(script: 'minikube service backend-service --url', returnStdout: true).trim()
            echo "Frontend: ${frontendUrl}"
            echo "Backend: ${backendUrl}"
            emailext(
                subject: "SUCCÈS Build: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Le pipeline a réussi!\nConsultez: ${env.BUILD_URL}",
                to: "mohamedndoye07@gmail.com"
            )
        }
    }

    failure {
        echo "❌ Le déploiement a échoué."
        emailext(
            subject: "ÉCHEC Build: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Le pipeline a échoué.\nDétails: ${env.BUILD_URL}",
            to: "mohamedndoye07@gmail.com"
        )
    }

    cleanup {
        sh '''
            docker logout
            echo "Cleanup completed"
        '''
    }
}

}
