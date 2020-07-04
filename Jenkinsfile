def label = "slave-${UUID.randomUUID().toString()}"

def helmLint(String chartDir) {
    sh "helm lint ${chartDir}"
}
def helmInit() {
    sh "helm init --client-only --stable-repo-url https://mirror.azure.cn/kubernetes/charts/"
}
def helmRepo(Map args) {
    sh "helm repo add --username ${args.username} --password ${args.password} myrepo https://registry.midland.com.cn/chartrepo"

    println "update repo"
    sh "helm repo update"
    println "fetch chart package"
    sh """
      helm fetch myrepo/library/hftpublic
      tar xzvf hftpublic-0.1.0.tgz
	  sed -i 's/hftpublic/${args.name}/g' ./hftpublic/Chart.yaml
	  sed -i 's/hftpublic/${args.name}/g' ./hftpublic/values.yaml
	  sed -i 's/hftpublic/${args.name}/g' ./hftpublic/templates/deployment.yaml
	  sed -i 's/hftpublic/${args.name}/g' ./hftpublic/templates/_helpers.tpl
	  sed -i 's/hftpublic/${args.name}/g' ./hftpublic/templates/service.yaml
    """
	
}
def helmDeploy(Map args) {
    helmInit()
    helmRepo(args)

    if (args.dry_run) {
    	sh "helm upgrade --install ${args.name} hftpublic  --set api.image.repository=${args.image} --set api.image.tag=${args.tag} --set replicaCount=4 --set service.namespaces=${args.namespace}"
        echo "应用 ${args.name} 部署成功. 可以使用 helm status ${args.name} 查看应用状态${args.namespace}"
    } else {
    	sh "helm upgrade --install ${args.name} hftpublic  --set api.image.repository=${args.image} --set api.image.tag=${args.tag} --set service.namespaces=${args.namespace}"
        echo "应用 ${args.name} 部署成功. 可以使用 helm status ${args.name} 查看应用状态${args.namespace}"
    }
}

podTemplate(label: label, containers: [
  containerTemplate(name: 'maven', image: 'registry.midland.com.cn/helm/mvn-jdk8:3.6.3', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'docker', image: 'registry.midland.com.cn/helm/docker', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'kubectl', image: 'registry.midland.com.cn/helm/kubectl', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'helm', image: 'registry.midland.com.cn/helm/helm', command: 'cat', ttyEnabled: true)

], volumes: [
  hostPathVolume(mountPath: '/root/.m2', hostPath: '/root/.m2'),
  hostPathVolume(mountPath: '/tmp/jenkins/.kube', hostPath: '/root/.kube'),
  hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
], serviceAccount: 'jenkins') {
  node(label) {
	def dockerRegistryUrl = "registry.midland.com.cn"
	def imageEndpoint = "chartrepo"
	def projname = env.JOB_NAME
	def image = "${dockerRegistryUrl}/${imageEndpoint}/${projname}"
	def imageTag = ""
	def myRepo = ""
	def isonline = false
	def spaces = env.myuserInput
	
	stage('Clone'){
			myRepo = checkout scm
			build_tag = myRepo.SVN_REVISION
			build_name = env.BUILD_NUMBER
			imageTag = "${build_name}-${build_tag}"
			script {
				if ( spaces == "online" ) {
					isonline = true
				}
			}

	}
    stage('maven') {
      container('maven') {
	  			sh "sed -i 's/node43.com/registry.midland.com.cn/g' ./Dockerfile"
				sh "sed -i 's/hft/helm/g' ./Dockerfile"  
      			sh "sed -i 's/server.port=[0-9]*/server.port=8080/g' ./src/main/resources/application*"
				sh "if [ ${myuserInput} == 'devcommon' ];then echo 'spring.profiles.active=dev' > ./src/main/resources/bootstrap.properties && sed -r -i  's/[0-9]{1,3}.[0-9]{1,3}.[0-9]{1,3}.[0-9]{1,3}:[0-9]{4}/hfteureka-server.dev/g' ./src/main/resources/bootstrap-* ;elif [ ${myuserInput} == 'graycommon' ];then echo 'spring.profiles.active=gray' > ./src/main/resources/bootstrap.properties && sed -r -i  's/[0-9]{1,3}.[0-9]{1,3}.[0-9]{1,3}.[0-9]{1,3}:[0-9]{4}/hfteureka-server.gray/g' ./src/main/resources/bootstrap-* ; else echo 'spring.profiles.active=${myuserInput}' > ./src/main/resources/bootstrap.properties ;fi"
				sh "if [ ${myuserInput} != 'devcommon' -a ${myuserInput} != 'graycommon' ];then sed -r -i  's/[0-9]{1,3}.[0-9]{1,3}.[0-9]{1,3}.[0-9]{1,3}:[0-9]{4}/hfteureka-server/g' ./src/main/resources/bootstrap-* ;fi"
				sh "mvn clean package -U  -Dmaven.test.skip=true -P linux"
      }
    }
    stage('docker') {
      container('docker') {
			  withCredentials([[$class: 'UsernamePasswordMultiBinding',
				credentialsId: 'dockerhub',
				usernameVariable: 'DOCKER_HUB_USER',
				passwordVariable: 'DOCKER_HUB_PASSWORD']]) {
				sh """
				  docker login ${dockerRegistryUrl} -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD}
				  docker build -t ${image}:${imageTag} .
				  docker push ${image}:${imageTag}
				  docker rmi ${image}:${imageTag}
				  """
			  }
			}
      }

    stage('deploy') {
	withCredentials([[$class: 'UsernamePasswordMultiBinding',
	credentialsId: 'dockerhub',
	usernameVariable: 'DOCKER_HUB_USER',
	passwordVariable: 'DOCKER_HUB_PASSWORD']]) {
	container('helm') {
		helmDeploy(
            dry_run     : isonline,
            name        : "${projname}",
            chartDir    : "${projname}",
            namespace   : "${myuserInput}",
            tag         : "${imageTag}",
            image       : "${image}",
            username    : "${DOCKER_HUB_USER}",
            password    : "${DOCKER_HUB_PASSWORD}"
        )
        echo "[INFO] 部署应用成功..."
		
      }
	  }

    }
  }
}
