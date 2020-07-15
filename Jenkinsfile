def docker_images = ["hejm37/torch-envs:10.1-cudnn7-devel-ubuntu18.04-pt1.3", "hejm37/torch-envs:10.2-cudnn7-devel-ubuntu18.04-pt1.5"]
def torch_versions = ["1.3.0", "1.5.0"]
def torchvision_versions = ["0.4.2", "0.6.0"]


def get_stages(docker_image) {
    def aliyun_mirror_args = "-i http://mirrors.aliyun.com/pypi/simple --trusted-host mirrors.aliyun.com"
    stages = {
        docker.image(docker_image).inside('-u root --gpus all') {
            try {
                stage("${docker_image}") {
                    sh "pwd"
                    sh "ls"
                    sh "echo 'Running in ${docker_image}'"
                }
                stage("${docker_image}-before_install") {
                    sh "apt-get update && apt-get install -y ninja-build"
                }
                stage("${docker_image}-install") {
                    sh "pip install Pillow==6.2.2 ${aliyun_mirror_args} # remove this line when torchvision>=0.5"
                    sh "pip install mmcv-full ${aliyun_mirror_args}"
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
                    sh "python setup.py build_ext --inplace"
                    sh "coverage run --branch --source mmdet -m py.test -v --xdoctest-modules tests mmdet"
                }

                // only if success
                sh "coverage report -m"
            } catch(e) {
                echo "Build failed for ${docker_image}"
                throw e
            }
        }
    }
    return stages
}


node('master') {
    // fetch latest change from SCM (Source Control Management)
    checkout scm

    def stages = [:]
    for (int i = 0; i < docker_images.size(); i++) {
        def docker_image = docker_images[i]
        def torch = torch_versions[i]
        def torchvision = torchvision_versions[i]
        def tag = docker_image + '_' + torch + '_' + torchvision
        stages[tag] = get_stages(docker_image)
    }
    parallel stages
}
