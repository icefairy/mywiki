#  cdh编写spark依赖问题

cdh的jar依赖不能直接用maven官方仓库的版本，需要使用cdh提供的仓库中的版本：

https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_cdh_62_maven_artifacts.html#concept_2gp_d8n_yk

否则会出现jar包不兼容的情况，并且cdh中自带的jar包在pom中dependency下的scope需要设置为provided：provided

