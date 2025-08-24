pipeline{
    agent any
	  stages{
	      stage('build'){
		        steps{
			          sh "docker build -t my-apache-app:${env.BUILD_NUMBER} ." 				 
}
    		}
		
		    stage('deploy'){
		        steps{
			    			sh "docker run -d -p 808${env.BUILD_NUMBER}:80 my-apache-app:${env.BUILD_NUMBER}"
						}
				}
	
	}
	
}
