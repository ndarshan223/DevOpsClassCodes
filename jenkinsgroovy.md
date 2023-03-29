Install jenkins

sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum upgrade
# Add required dependencies for the jenkins package
sudo yum install java-11-openjdk
sudo yum install jenkins
sudo systemctl daemon-reload
systemctl restart jenkins
systemctl status jenkins
====================================================================================================================================================

Install maven and git to build java project on jenkins server

yum install git -y 
wget https://mirrors.estointernet.in/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
sudo mkdir -p /opt/maven
sudo tar -xvzf apache-maven-3.6.3-bin.tar.gz -C /opt/maven/ --strip-components=1
sudo ln -s /opt/maven/bin/mvn /usr/bin/mvn
mvn --version    

clean package
====================================================================================================================================================
Install docker on same host as jenkins

sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
sudo systemctl start docker
sudo systemctl enable docker
systemctl status docker
chown root:jenkins /var/run/docker.sock
systemctl daemon-reload
systemctl restart docker
systemctl status docker

====================================================================================================================================================

CI pipeline using git and maven

pipeline
{
	agent any
	stages
	{
		stage('Checkout')
		{
			steps
			{
				git 'https://github.com/jsachdev07/DevOpsClassCodes.git'
			}
		}
		
		stage('Compile')
		{
			steps
			{
				sh 'mvn compile'
			}
		}

		stage('Test')
		{
			steps
			{
				sh 'mvn test'
			}
		}

		stage('Build')
		{
			steps
			{
				sh 'mvn package'
			}
		}
     }
}
====================================================================================================================================================

CI-CD pipeline using docker installed on jenkins host

Extra manual steps required are -->
1. install docker pipeline plugin on jenkins server
2. add the credentials for your DockerHub account in the jenkins server with id as dockerhub

pipeline
{
	agent any
	stages
	{
		stage('Checkout')
		{
			steps
			{
				git 'https://github.com/jsachdev07/DevOpsClassCodes.git'
			}
		}
		
		stage('Compile')
		{
			steps
			{
				sh 'mvn compile'
			}
		}

		stage('Test')
		{
			steps
			{
				sh 'mvn test'
			}
		}

		stage('Build')
		{
			steps
			{
				sh 'mvn package'
			}
		}
		
		stage('Build Docker Image')
		{
			steps
			{
			    sh 'cd /var/lib/jenkins/workspace/$JOB_NAME/'
			    sh 'cp /var/lib/jenkins/workspace/$JOB_NAME/target/addressbook.war /var/lib/jenkins/workspace/$JOB_NAME/'
				sh 'docker build -t addressbook:$BUILD_NUMBER .'
				sh 'docker tag addressbook:$BUILD_NUMBER jsachdev07/addressbook:$BUILD_NUMBER'
			}
		}

		stage('Push Docker Image')
		{ 
			steps
			{   
			    withDockerRegistry([ credentialsId: "dockerhub", url: "" ])
			    {
			       sh 'docker push jsachdev07/addressbook:$BUILD_NUMBER'
			    }
			}
		}

		stage('Deploy as container')
		{
			steps
			{
				sh 'docker run -itd -P jsachdev07/addressbook:$BUILD_NUMBER'
			}
		}   
	}
		
}

====================================================================================================================================================
CI-CD pipeline using docker host running on different machine other then jenkins host

Extra steps that needs to be performed are -->

1. create another machine and install docker on it
2. after installing docker open port on docker for communication from remote machine

vi /lib/systemd/system/docker.service --> edit below line and open port 4243
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:4243 -H unix:///var/run/docker.sock

3. systemctl daemon-reload 
   systemctl restart docker

In the pipeline provide -H dockerIP:4243 in the all the docker commands

pipeline
{
	agent any
	stages
	{
		stage('Checkout')
		{
			steps
			{
				git 'https://github.com/jsachdev07/DevOpsClassCodes.git'
			}
		}
		
		stage('Compile')
		{
			steps
			{
				sh 'mvn compile'
			}
		}

		stage('Test')
		{
			steps
			{
				sh 'mvn test'
			}
		}

		stage('Build')
		{
			steps
			{
				sh 'mvn package'
			}
		}
		
		stage('Build Docker Image')
		{
			steps
			{
			    sh 'cd /var/lib/jenkins/workspace/$JOB_NAME/'
			    sh 'cp /var/lib/jenkins/workspace/$JOB_NAME/target/addressbook.war /var/lib/jenkins/workspace/$JOB_NAME/'
				sh 'docker -H 30.30.7.4:4243 build -t addressbook:$BUILD_NUMBER .'
				sh 'docker -H 30.30.7.4:4243 tag addressbook:$BUILD_NUMBER jsachdev07/addressbook:$BUILD_NUMBER'
			}
		}

		stage('Push Docker Image')
		{ 
			steps
			{   
			    withDockerRegistry([ credentialsId: "dockerhub", url: "" ])
			    {
			       sh 'docker -H 30.30.7.4:4243 push username/addressbook:$BUILD_NUMBER'
			    }
			}
		}

		stage('Deploy as container')
		{
			steps
			{
				sh 'docker -H 30.30.7.4:4243 run -itd -P username/addressbook:$BUILD_NUMBER'
			}
		}

	}
		
}
