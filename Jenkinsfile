pipeline {
    agent any
	stages {
	    stage('Build Project') {
	        steps {
                // Build the api blog project
                script {
                    sh '''
                        cd backend/blogs
                        npm install
                        npm run build
                    '''
                }
                // Frontend build
                script {
                    sh '''
                        cd frontend
                        npm install
                        npm run build
                    '''
                }
	        }
	    }
	    stage("Unit Test") {
	        steps {
				// Frontend unit test execution,
	             script {
	                sh '''
	                    cd frontend
	                    npm run test
	                '''
	             }
	        }
	    }
	    stage('Run Linting') {
	        steps {
                // Run linting the api blog project
                script {
                    sh '''
                        cd backend/blogs
                        npm run lint
                    '''
	            }
	            // Run linting the api blog project
	            script {
	            sh '''
                    cd frontend
                    npm run lint
	            '''
	             }
	        }
	    }
	    stage("Dependencies analysis") {
            steps {
                script {
                     sh '''
                         cd backend/blogs
                         npm install
                         npm audit fix --production --audit-level=critical --force
                         npm audit fix --production --force
                         npm audit --production --audit-level=critical
                     '''
                }
				script {
					sh '''
						cd frontend
                        npm install
                        npm audit fix --production --audit-level=critical --force
                        npm audit fix --production --force
                        npm audit --production --audit-level=critical
					'''
				}
            }
        }
		stage("Run Migrations") {
			steps {
				script {
					Throwable caughtException = null

					catchError(buildResult: 'SUCCESS', stageResult: 'ABORTED') {
						try {
							sh '''
								cd backend/blogs
								npm install
								npm run migrations:run -- -d dist/infrastructure/database/database.source.js
							'''

						} catch (Throwable e) {
							sh '''
								cd backend/blogs
								npm install
								npm run migrations:revert -- -d dist/infrastructure/database/database.source.js
							'''
							caughtException = e
						}
					}

					if (caughtException) {
						error caughtException.message
					}
				}
			}
		}
		stage("Pushing docker images to ECR") {
			steps {
				script {
					withAWS(credentials: 'aws-credentials', region: 'eu-west-2') {
						sh '''
							cd backend/blogs
							aws ecr get-login-password --region eu-west-2 | docker login --username AWS --password-stdin 622817277005.dkr.ecr.eu-west-2.amazonaws.com
							docker build -t backend-blogs:v1 .
							docker tag backend-blogs:v1 622817277005.dkr.ecr.eu-west-2.amazonaws.com/backend-blogs:v1
							docker push 622817277005.dkr.ecr.eu-west-2.amazonaws.com/backend-blogs:v1
						'''
						sh '''
							cd frontend
							aws ecr get-login-password --region eu-west-2 | docker login --username AWS --password-stdin 622817277005.dkr.ecr.eu-west-2.amazonaws.com
							docker build -t frontend:v1 .
							docker tag frontend:v1 622817277005.dkr.ecr.eu-west-2.amazonaws.com/frontend:v1
							docker push 622817277005.dkr.ecr.eu-west-2.amazonaws.com/frontend:v1
						'''
						sh '''
							cd reverse-proxy
							aws ecr get-login-password --region eu-west-2 | docker login --username AWS --password-stdin 622817277005.dkr.ecr.eu-west-2.amazonaws.com
							docker build -t reverse-proxy:v1 .
							docker tag reverse-proxy:v1 622817277005.dkr.ecr.eu-west-2.amazonaws.com/reverse-proxy:v1
							docker push 622817277005.dkr.ecr.eu-west-2.amazonaws.com/reverse-proxy:v1
						'''
					}
				}
			}
		}
		stage("Orchestrate docker images") {
			steps {
				script {
					withAWS(credentials: 'aws-credentials', region: 'eu-west-2') {
						Throwable caughtException = null

						catchError(buildResult: 'SUCCESS', stageResult: 'ABORTED') {
							try {
								sh '''
									aws eks update-kubeconfig --region eu-west-2 --name capstone-cluster
									kubectl apply -f deployments/backend-blogs-deployment.yaml
									kubectl apply -f deployments/frontend-deployment.yaml
									kubectl apply -f deployments/reverse-proxy-deployment.yaml
									kubectl apply -f deployments/backend-blogs-service.yaml
									kubectl apply -f deployments/frontend-service.yaml
									kubectl apply -f deployments/reverse-proxy-service.yaml
									kubectl expose deployment frontend --type=LoadBalancer --name=publicfrontend
									kubectl expose deployment reverse-proxy --type=LoadBalancer --name=publicreverseproxy
								'''
							} catch (Throwable e) {
								sh '''
									aws eks update-kubeconfig --region eu-west-2 --name capstone-cluster
									kubectl delete all --all
								'''
								caughtException = e
							} finally {
								sh '''
									aws eks update-kubeconfig --region eu-west-2 --name capstone-cluster
									kubectl get services
								'''
								sh '''
									aws eks update-kubeconfig --region eu-west-2 --name capstone-cluster
									kubectl get deployments
								'''
								sh '''
									aws eks update-kubeconfig --region eu-west-2 --name capstone-cluster
									kubectl get pods
								'''
							}
							if (caughtException) {
								error caughtException.message
							}
						}

					}
				}
			}
		}
	}
}