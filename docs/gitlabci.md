## gitlab ci 部署应用骚操作

### 背景

完整的gitlab ci一般都有`stage`用于部署应用。

部分应用没有采用docker方式部署的话，怎么可以快速通过ci串联部署？

### 骚操作

简述：用ssh的方式进行ci控制和部署。

最终部署可以通过`supervisor的方式`进行部署，或者直接放后台。

#### 环境介绍

变量名称|备注
---|---
TARGET_IP|部署机器的IP
TARGET_SSH_PORT|部署机器ssh的端口
TARGET_SSH_USER|部署机器ssh的用户名
var_id_rsa|部署机器TARGET_SSH_USER用户的私钥变量，需要以前将私钥存到gitlab ci的环境配置变量里面


#### `.gitlab-ci.yml`代码片段

```
variables:
  TARGET_IP: 1.2.3.4
  TARGET_SSH_USER: user
  TARGET_SSH_PORT: 22

stages:
  - build
  - deploy


deploy_target:
  stage: deploy
  only:
    - <branch>
  script:
    - PATH_IDRSA=`find . -name 'id_rsa'`
    - echo -n "${var_id_rsa}" > ${PATH_IDRSA}
    - echo "check newworks connection."
    - bash -c "ping -c 1 ${TARGET_IP}"
    - REPO=`echo $CI_PROJECT_DIR | awk -F '/' '{print $NF}'`
    - GROUP=`echo $CI_PROJECT_DIR | awk -F '/' '{print $(NF-1)}'`
    - DOCKER_REPO="$GROUP/$REPO"
    # change keys Permissions 
    - bash -c "chmod 600 ${PATH_IDRSA}"
    # clear previous dir (Be Cautious!!)
    - bash -c "echo `ssh -i ${PATH_IDRSA} -p ${TARGET_SSH_PORT} -o StrictHostKeyChecking=no ${TARGET_SSH_USER}@${TARGET_IP} \"rm -rf ~/gitlab_projects/${DOCKER_REPO} 2> /dev/null\"`"
    # mkdir for scp code
    - bash -c "echo `ssh -i ${PATH_IDRSA} -p ${TARGET_SSH_PORT} -o StrictHostKeyChecking=no ${TARGET_SSH_USER}@${TARGET_IP} \"mkdir -p ~/gitlab_projects/${DOCKER_REPO}\"`"
    - CODE_ROOT_PATH=`dirname $(find . -name '.gitlab-ci.yml')`
    # scp code
    # first tar zip the code
    - mkdir -p /tmp && cd ${CODE_ROOT_PATH} && tar -czvf /tmp/${REPO}.tar.gz . && mv /tmp/${REPO}.tar.gz .
    - bash -c "scp -r -i ${PATH_IDRSA} -P ${TARGET_SSH_PORT} -o StrictHostKeyChecking=no ${CODE_ROOT_PATH}/${REPO}.tar.gz ${TARGET_SSH_USER}@${TARGET_IP}:/home/${TARGET_SSH_USER}/gitlab_projects/${DOCKER_REPO}"
    # unzip
    - CICD_COMMANDS="cd /home/${TARGET_SSH_USER}/gitlab_projects/${DOCKER_REPO} && tar -xzvf ${REPO}.tar.gz"
    - ssh -i ${PATH_IDRSA} -p ${TARGET_SSH_PORT} -o StrictHostKeyChecking=no ${TARGET_SSH_USER}@${TARGET_IP} "${CICD_COMMANDS}"
    - CICD_COMMANDS="whoami"
    - bash -c "echo `ssh -i ${PATH_IDRSA} -p ${TARGET_SSH_PORT} -o StrictHostKeyChecking=no ${TARGET_SSH_USER}@${TARGET_IP} \"${CICD_COMMANDS}\"`"
```