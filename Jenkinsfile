def docker_images = ["hejm37/python-envs:cuda10.1-cudnn7-devel-ubuntu18.04-py37", "hejm37/python-envs:cuda10.1-cudnn7-devel-ubuntu18.04-py36"]
def torch_versions = ["1.3.1", "1.5.0"]
def torchvision_versions = ["0.4.2", "0.6.0"]
def cuda_archs = ["6.0", "7.0"]

def setBuildStatus(message, state, tag) {
  step([
      $class: "GitHubCommitStatusSetter",
      reposSource: [$class: "ManuallyEnteredRepositorySource", url: env.GIT_URL],
      contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "ci/jenkins/build-status/${tag}"],
      errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
      statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]] ]
  ]);
}

void setBuildStatus_(String message, String state) {
  step([
      $class: "GitHubCommitStatusSetter",
      reposSource: [$class: "ManuallyEnteredRepositorySource", url: "https://github.com/my-org/my-repo"],
      contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "ci/jenkins/build-status"],
      errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
      statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]] ]
  ]);
}

def get_stages(docker_image, torch, torchvision, cuda_arch) {
    def tag = docker_image + '_' + torch + '_' + torchvision + '_' + cuda_arch
    def aliyun_mirror_args = "-i http://mirrors.aliyun.com/pypi/simple --trusted-host mirrors.aliyun.com"
    stages = {
        docker.image(docker_image).inside('-u root --gpus all') {
            try {
                // githubNotify description: 'This is a shorted example',  status: 'PENDING'

                stage("${docker_image}") {
                    sh "echo 'Running in ${docker_image}'"
                }
                stage("${docker_image}-before_install") {
                    sh "apt-get install -y ninja-build"
                }
                stage("${docker_image}-install") {
                    sh "pip install Pillow==6.2.2 ${aliyun_mirror_args} # remove this line when torchvision>=0.5"
                    sh "pip install torch==${torch} torchvision==${torchvision} ${aliyun_mirror_args}"
                    sh "pip install mmcv-nightly ${aliyun_mirror_args}"
                    sh "pip install 'git+https://github.com/cocodataset/cocoapi.git#subdirectory=PythonAPI' ${aliyun_mirror_args}"
                    sh "pip install -r requirements.txt ${aliyun_mirror_args}"
                }
                stage("${docker_image}-before_script") {
                    sh "flake8 ."
                    sh "isort -rc --check-only --diff mmdet/ tools/ tests/"
                    sh "yapf -r -d --style .style.yapf mmdet/ tools/ tests/ configs/"
                }
                stage("${docker_image}-script") {
                    sh "python setup.py check -m -s"
                    sh "TORCH_CUDA_ARCH_LIST=${cuda_arch} python setup.py build_ext --inplace"
                    sh "coverage run --branch --source mmdet -m py.test -v --xdoctest-modules tests mmdet"
                }

                // only if success
                sh "coverage report -m"
                // githubNotify description: 'This is a shorted example',  status: 'SUCCESS'
                setBuildStatus("Build succeeded", "SUCCESS", "${tag}")
                // setBuildStatus_("Build succeeded", "SUCCESS")
            } catch(e) {
                echo "Build failed for ${tag}"
                // githubNotify description: 'This is a shorted example',  status: 'FAILURE'
                setBuildStatus("Build failed", "FAILURE", "${tag}")
                // setBuildStatus_("Build failed", "FAILURE")
                throw e
            }
        }
    }
    return stages
}

node('master') {
    def stages = [:]
    for (int i = 0; i < docker_images.size(); i++) {
        def docker_image = docker_images[i]
        for (int j = 0; j < torch_versions.size(); j++) {
            def torch = torch_versions[j]
            def torchvision = torchvision_versions[j]
            def cuda_arch = cuda_archs[j]
            def tag = docker_image + '_' + torch + '_' + torchvision + '_' + cuda_arch
            stages[tag] = get_stages(docker_image, torch, torchvision, cuda_arch)
        }
    }

    checkout scm

    parallel stages

}
