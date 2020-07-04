def docker_images = ["hejm37/python-envs:cuda10.1-cudnn7-devel-ubuntu18.04-py37", "hejm37/python-envs:cuda10.1-cudnn7-devel-ubuntu18.04-py36"]
def torch_versions = ["1.3.1", "1.5.0"]
def torchvision_versions = ["0.4.2", "0.6.0"]
def cuda_archs = ["6.0", "7.0"]

def get_stages(docker_image, env_torch, env_torchvision, env_cuda_arch) {
    stages = {
        docker.image(docker_image).inside('-u root --gpus all') {
            try {
                stage("${docker_image}") {
                    sh "echo 'Running in ${docker_image}'"
                }
                stage("${docker_image}-before_install") {
                    sh "apt-get install -y ninja-build"
                }
                stage("${docker_image}-install") {
                    sh "pip install Pillow==6.2.2  # remove this line when torchvision>=0.5"
                    sh "pip install torch==${env_torch} torchvision==${env_torchvision}"
                    sh "pip install mmcv-nightly"
                    sh "pip install 'git+https://github.com/cocodataset/cocoapi.git#subdirectory=PythonAPI'"
                    sh "pip install -r requirements.txt"
                }
                stage("${docker_image}-before_script") {
                    sh "flake8 ."
                    sh "isort -rc --check-only --diff mmdet/ tools/ tests/"
                    sh "yapf -r -d --style .style.yapf mmdet/ tools/ tests/ configs/"
                }
                stage("${docker_image}-script") {
                    sh "python setup.py check -m -s"
                    sh "TORCH_CUDA_ARCH_LIST=${env_cuda_arch} python setup.py build_ext --inplace"
                    sh "coverage run --branch --source mmdet -m py.test -v --xdoctest-modules tests mmdet"
                }

                // only if success
                sh "coverage report -m"
            } catch(e) {
                echo "Build failed for ${docker_image}_${env_torch}_${env_torchvision}_${env_cuda_arch}"
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

    parallel stages

}
