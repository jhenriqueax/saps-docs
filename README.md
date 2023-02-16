# SAPS
* Verifique de estar usando Ubuntu 16.04 ou 18.04


# Componentes SAPS
1. [Common](#common)
1. [Catalog](#catalog)
1. [Archiver](#archiver)
1. [Dispatcher](#dispatcher)
1. [Scheduler](#scheduler)
1. [Dashboard](#dashboard)
1. [Arrebol](#arrebol)
    1. [Clean Option](#clean-option)
1. [Workers](#workers)
1. [Testes NOP](#testes-nop)
1. [Crontab](#crontab)
1. [Logrotate](#logrotate)


-------------------------------------------------------------------
## [Common](https://github.com/ufcg-lsd/saps-common)
* Esse repositório é necessário para:
    1. [Catalog](#catalog)
    2. [Archiver](#archiver)
    3. [Dispatcher](#dispatcher)
    4. [Scheduler](#scheduler)

### Instalação:
1. Instale o JDK, Maven, Git
    ```
    sudo apt-get update
    sudo apt-get -y install openjdk-8-jdk
    sudo apt-get -y install maven
    sudo apt-get -y install git
    ```
2. Clone e instale as dependencias
    ```
    git clone https://github.com/ufcg-lsd/saps-common ~/saps-common
    cd ~/saps-common
    sudo mvn install 
    ```

-------------------------------------------------------------------
## [Catalog](https://github.com/ufcg-lsd/saps-catalog)
### Variaveis a serem definidas:
* $catalog_user=catalog_user
* $catalog_passwd=catalog_passwd
* $catalog_db_name=catalog_db_name
* $installed_version= Verifique a sua versão do PostgreSQL 

### Instalação:
1. Configure o [saps-common](#common)
1. Clone e instale as dependencias
    ```
    git clone https://github.com/ufcg-lsd/saps-catalog ~/saps-catalog
    cd ~/saps-catalog
    sudo mvn install 
    ```
1. Instale o postgres 
    ``` 
    sudo apt-get install -y postgresql
    ```
1. Instale o pip e o pandas
    ``` 
    sudo apt install python3-pip
    pip3 install pandas
    pip3 install tqdm
    ```
1. Configure o Catalog
    ``` 
    sudo su postgres
    export catalog_user=catalog_user
    export catalog_passwd=catalog_passwd
    export catalog_db_name=catalog_db_name
    psql -c "CREATE USER $catalog_user WITH PASSWORD '$catalog_passwd';"
    psql -c "CREATE DATABASE $catalog_db_name OWNER $catalog_user;"
    psql -c "GRANT ALL PRIVILEGES ON DATABASE $catalog_db_name TO $catalog_user;"
    exit
    ```
1. Adicione a senha como exigencia para acesso ao PostgreSQL 
    * Esse passo irá fazer com que qualquer usuário cadastrado no postgresql precise de uma senha para acessar o banco de dados
    ```
    sudo su
    export installed_version=`ls /etc/postgresql`
    sed -i 's/peer/md5/g' /etc/postgresql/$installed_version/main/pg_hba.conf
    echo "host all all 0.0.0.0/0 md5" >> /etc/postgresql/$installed_version/main/pg_hba.conf
    sed -i "$ a\listen_addresses = '*'" /etc/postgresql/$installed_version/main/postgresql.conf
    service postgresql restart
    exit
    ```
1. Teste o acesso do Catalog
    ```
    psql -h <catalog_ip_address> -p 5432 $catalog_db_name $catalog_user
    ```
    * Exemplo:
        ```
        psql -h localhost -p 5432 catalog_db_name catalog_user
        ```

### Configuração:
* Execute o script **/scripts/fetch_landsat_data.sh** (ele demora um pouco)
```
cd ~/saps-catalog/scripts
sudo bash fetch_landsat_data.sh
```

-------------------------------------------------------------------


## [Archiver](https://github.com/ufcg-lsd/saps-archiver)
### Variaveis a serem definidas:
* $nfs_server_folder_path=/nfs

### Instalação:
1. Configure o [saps-common](#common)
2. Instale as dependencias do [saps-catalog](#catalog)
    ```
    git clone https://github.com/ufcg-lsd/saps-catalog ~/temp/saps-catalog
    cd ~/temp/saps-catalog
    sudo mvn install 
    cd -
    sudo rm -rf ~/temp/saps-catalog
    sudo rm -d ~/temp/
    ```
3. Clone e instale as dependencias
    ```
    git clone https://github.com/ufcg-lsd/saps-archiver ~/saps-archiver
    cd ~/saps-archiver
    sudo mvn install 
    ```
4. Configure uma pasta compartilhada
* Crie uma pasta nfs para armazenar os arquivos de cada etapa de processamento
```
    mkdir -p /nfs
```

5. Configure apache (necessário para acesso aos dados por email) 
* (Seria legal migrar isso pra nginx)
* Instale o apache
  ```
  sudo apt-get install -y apache2
  ```

* Modifique o arquivo sites-available/default-ssl.conf
  ```
  sudo vim /etc/apache2/sites-available/default-ssl.conf
  ```
  * Mude o DocumentRoot para o diretorio do nfs (default = /nfs)
    ```
    DocumentRoot $nfs_server_folder_path 
    # Exemplo: DocumentRoot /nfs
    ```

* Modifique o arquivo sites-available/000-default.conf
  ```
  sudo vim /etc/apache2/sites-available/000-default.conf
  ```
  * Mude o DocumentRoot e adicione as linhas em sequencia
    ```
    DocumentRoot $nfs_server_folder_path 
    # Exemplo: DocumentRoot /nfs
            Options +Indexes
            <Directory $nfs_server_folder_path>
            # Exemplo: <Directory /nfs>
                    Options Indexes FollowSymLinks
                    AllowOverride None
                    Require all granted
            </Directory>
    ```


* Modifique o arquivo sites-available/000-default.conf
  ```
  sudo vim /etc/apache2/apache2.conf
  ```
  * Mude o FilesMatch 
    ```
    <FilesMatch ".+\.(txt|TXT|nc|NC|tif|TIF|tiff|TIFF|csv|CSV|log|LOG|metadata)$">
            ForceType application/octet-stream
            Header set Content-Disposition attachment
    </FilesMatch>
    ```

* Após configurar os arquivos, execute:
  ```
  sudo a2enmod headers
  sudo service apache2 restart
  ```

### Configuração:
Configure o arquivo /config/archiver.conf de acordo com os outros componentes
* Exemplo (nfs): [archiver.conf](./confs/archiver/clean/archiver.conf) 

### Execução:
* Executando archiver
    ```
    bash bin/start-service
    ```

* Parando archiver
    ```
    bash bin/stop-service
    ```

------------------------------------------------------------------
## [Dispatcher](https://github.com/ufcg-lsd/saps-dispatcher)
### Instalação:
1. Configure o [saps-common](#common)
2. Instale as dependencias do [saps-catalog](#catalog)
    ```
    git clone https://github.com/ufcg-lsd/saps-catalog ~/temp/saps-catalog
    cd ~/temp/saps-catalog
    sudo mvn install 
    cd -
    sudo rm -rf ~/temp/saps-catalog
    sudo rm -d ~/temp/
    ```
3. Clone e instale as dependencias
    ```
    git clone https://github.com/ufcg-lsd/saps-dispatcher ~/saps-dispatcher
    cd ~/saps-dispatcher
    sudo mvn install 
    ```
4. Instale as dependências do script python (get_wrs.py)
    ```
    sudo apt-get install -y python-gdal
    sudo apt-get install -y python-shapely
    sudo apt-get -y install curl jq sed
    ```

### Configuração:
Configure o arquivo **/config/dispatcher.conf** de acordo com os outros componentes
* Exemplo (nfs): [dispatcher.conf](./confs/dispatcher/clean/dispatcher.conf) 

### Execução:
* Executando dispatcher
    ```
    bash bin/start-service
    ```

* Parando dispatcher
    ```
    bash bin/stop-service
    ```

-------------------------------------------------------------------
## [Scheduler](https://github.com/ufcg-lsd/saps-scheduler)
### Instalação:
1. Configure o [saps-common](#common)
2. Instale as dependencias do [saps-catalog](#catalog)
    ```
    git clone https://github.com/ufcg-lsd/saps-catalog ~/temp/saps-catalog
    cd ~/temp/saps-catalog
    sudo mvn install 
    cd -
    sudo rm -rf ~/temp/saps-catalog
    sudo rm -d ~/temp/
    ```
3. Clone e instale as dependencias
    ```
    git clone https://github.com/ufcg-lsd/saps-scheduler ~/saps-scheduler
    cd ~/saps-scheduler
    sudo mvn install 
    ```
### Configuração:
Configure o arquivo **/config/scheduler.conf** de acordo com os outros componentes
* Exemplo (nfs): [scheduler.conf](./confs/scheduler/clean/scheduler.conf) 

### Execução:
* Executando scheduler
    ```
    bash bin/start-service
    ```

* Parando scheduler
    ```
    bash bin/stop-service
    ```

-------------------------------------------------------------------
## [Dashboard](https://github.com/ufcg-lsd/saps-dashboard)
### Instalação:
1. Instale o curl e o nodejs
    ```
    sudo apt-get update
    sudo apt-get install -y curl
    curl -sL https://deb.nodesource.com/setup_13.x | sudo -E bash -
    sudo apt-get install -y nodejs
    ```
2. Clone e instale as dependencias
    ```
    git clone https://github.com/ufcg-lsd/saps-dashboard ~/saps-dashboard
    cd ~/saps-dashboard
    npm install
    ```

### Configuração Geral:
* Configure o host e as portas em [**/backend.config**](./confs/dashboard/clean/backend.config)
* Configure a urlSapsService em [**/public/dashboardApp.js**](./confs/dashboard/clean/dashboardApp.js) (Linha 52)


### Configuração do OAuth2 do EGI (opcional)
* O OAuth2 EGI serve para que usuários cadastrados no [EGI](https://aai.egi.eu/registry/auth/login/) possam se autenticar


1. **app.js**
  * É preciso configurar os [endpoints](https://docs.egi.eu/providers/check-in/sp/#endpoints) do check-in EGI no [**/app.js**](./confs/dashboard/clean/app.js) (Linhas 75 ~ 89)
  * Os endpoints de redirecionamento variam de acordo com os ambientes (desenvolvimento e produção) e precisam de algumas credenciais do usuário 
    * (Quem disponibilizou as credenciais pra gente foi o pessoal da europa)
    * Em ambiente de produção é preciso descomentar as linhas 159~162

2. **/public/dashboardApp.js**
  * Configure a EGISecretKey em [**/public/dashboardApp.js**](./confs/dashboard/clean/dashboardApp.js) (Linha 53)

3. **/public/services/dasboardService.js**
  * Configure a rota de callback do checkin EGI 
    * O redirecionamento do OAuthEGI precisa ser para um dominio autorizado, dentre eles, temos:
    * Desenvolvimento: 
      1. http://localhost:8081/auth-egi-callback
      1. https://saps-test.lsd.ufcg.edu.br/auth-egi-callback
    * Produção
      1. https://saps.vm.fedcloud.eosc-synergy.eu/auth-egi-callback

4. Observações
  * Caso esteja usando um dominio https, é preciso configurar o certificado SSL e o direcionamento das requisições.
    * Para isso, configure o nginx e o [certificado](https://certbot.eff.org/) (Siga o exemplo do arquivo **/nginx/default**)

### Execução:
* Executando dashboard
    ```
    sudo bash bin/start-dashboard
    ```

* Parando dashboard
    ```
    sudo bash bin/stop-dashboard
    ```

-------------------------------------------------------------------
## [Arrebol](https://github.com/ufcg-lsd/arrebol) 
### ***Clean Option***
### Variaveis a serem definidas:
* $arrebol_db_passwd=@rrebol
* $arrebol_db_name=arrebol
* $arrebol_db_user=arrebol_db_user

### Instalação:
1. Instale o JDK, Maven e Git
    ```
    sudo apt-get update
    sudo apt-get -y install openjdk-8-jdk
    sudo apt-get -y install maven
    sudo apt-get -y install git
    sudo apt-get install -y postgresql
    ```
1. Instale o ansible 
    ```
    sudo apt update
    sudo apt install --y software-properties-common
    sudo apt-add-repository --yes --update ppa:ansible/ansible
    sudo apt install -y ansible
    ```
1. Clone e instale as dependencias
    ```
    git clone -b develop https://github.com/cilasmarques/arrebol ~/arrebol
    cd ~/arrebol
    sudo mvn install
    ```
1. Configure o BD do arrebol
    ``` 
    sudo su postgres
    export arrebol_db_user=arrebol_db_user
    export arrebol_db_passwd=@rrebol
    export arrebol_db_name=arrebol 
    psql -c "CREATE USER $arrebol_db_user WITH PASSWORD '$arrebol_db_passwd';"
    psql -c "CREATE DATABASE $arrebol_db_name OWNER $arrebol_db_user;"
    psql -c "ALTER USER $arrebol_db_user PASSWORD '$arrebol_db_passwd';"
    exit
    ```

1. Teste o acesso do bd do arrebol
    ```
    psql -h <arrebol_ip_address> -p 5432 $arrebol_db_name arrebol_db_user
    ```
    * Exemplo:
        ```
        psql -h localhost -p 5432 arrebol arrebol_db_user
        ```


### Configuração:
Configure os arquivos **src/main/resources/application.properties** e **src/main/resources/arrebol.json** de acordo com os outros componentes
* Exemplo: [application.properties](./confs/arrebol/clean/application.properties) 
* Exemplo: [arrebol.json](./confs/arrebol/clean/arrebol.json) 

### Antes de executar, configure os workers do arrebol 
* Essa configuração deve ser feita na **mesma máquina que executará** o arrebol**.
* Para configurar o worker, siga esses [passos](#workers)

### Execução:
* Executando arrebol
    ```
    sudo bash bin/start-service.sh
    ```

* Parando arrebol
    ```
    sudo bash bin/stop-service.sh
    ```

### Configuração das tabelas do arrebol_db
1. Após a execução do arrebol, são criadas as tabelas no bd, com isso é preciso adicionar as seguintes constraints
    ```
    psql -h localhost -p 5432 arrebol postgres
    ALTER TABLE task_spec_commands DROP CONSTRAINT fk7j4vqu34tq49sh0hltl02wtlv;
    ALTER TABLE task_spec_commands ADD CONSTRAINT commands_id_fk FOREIGN KEY (commands_id) REFERENCES command(id) ON DELETE CASCADE;

    ALTER TABLE task_spec_commands DROP CONSTRAINT fk9y8pgyqjodor03p8983w1mwnq;
    ALTER TABLE task_spec_commands ADD CONSTRAINT task_spec_id_fk FOREIGN KEY (task_spec_id) REFERENCES task_spec(id) ON DELETE CASCADE;

    ALTER TABLE task_spec_requirements DROP CONSTRAINT fkrxke07njv364ypn1i8b2p6grm;
    ALTER TABLE task_spec_requirements ADD CONSTRAINT task_spec_id_fk FOREIGN KEY (task_spec_id) REFERENCES task_spec(id) ON DELETE CASCADE;

    ALTER TABLE task DROP CONSTRAINT fk303yjlm5m2en8gknk80nkd27p; 
    ALTER TABLE task ADD CONSTRAINT task_spec_id_fk FOREIGN KEY (task_spec_id) REFERENCES task_spec(id) ON DELETE CASCADE;
    ```

### Checagem
* Requisição
    ```
    curl http://127.0.0.1:8080/queues/default
    ```
* Resposta esperada
    ```
    {"id":"default","name":"Default Queue","waiting_jobs":0,"worker_pools":1,"pools_size":5}
    ```

-------------------------------------------------------------------
## Workers
### Configuração:
1. Instale Docker
    ```
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo apt-key fingerprint 0EBFCD88
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    sudo apt-get update
    sudo apt-get install -y docker-ce docker-ce-cli containerd.io
    ```

### Configurando worker para arrebol
* Modifique o arquivo /lib/systemd/system/docker.service
  ```
  sudo vim /lib/systemd/system/docker.service
  ```
  * Mude o ExecStart e substitua pelas seguintes linhas 
    ```
      ExecStart=/usr/bin/dockerd -H fd:// -H=tcp://0.0.0.0:5555 --containerd=/run/containerd/containerd.sock
    ```

* Reinicie o deamon e o docker
  ```
    sudo systemctl daemon-reload
    sudo service docker restart
  ```

* Faça pull imagens dockers
  ```
    sudo docker pull fogbow/inputdownloader:googleapis
    sudo docker pull fogbow/preprocessor:default
    sudo docker pull fogbow/worker:ufcg-sebal

    sudo docker pull fogbow/inputdownloader:usgsapis
    sudo docker pull fogbow/preprocessor:legacy
    sudo docker pull fogbow/worker:sebkc-sebal
    sudo docker pull fogbow/worker:sebkc-tseb
  ```

### Checagem
* Requisição
    ```
    curl http://127.0.0.1:5555/version
    ```
* Resposta esperada
    ```
    {"id":"default","name":"Default Queue","waiting_jobs":0,"worker_pools":1,"pools_size":5}
    ```

-------------------------------------------------------------------
## Testes NOP
### Adicione as tags dos testes NOP nas configurações dos seguintes componentes
1. [Dashboard](#-dashboard)
    * Arquivo: [dashboardApp.js](https://github.com/ufcg-lsd/saps-dashboard/blob/develop/public/dashboardApp.js)
    * Exemplo: [dashboardApp.js](./confs/dashboard/clean/dashboardApp.js) 
1. [Dispatcher](#dispatcher)
    * Arquivo: [execution_script_tags.json](https://github.com/ufcg-lsd/saps-dispatcher/blob/develop/resources/execution_script_tags.json)
    * Exemplo: [dispatcher.conf](./confs/dispatcher/clean/dispatcher.conf)
1. [Scheduler](#scheduler)
    * Arquivo: [execution_script_tags.json](https://github.com/ufcg-lsd/saps-scheduler/blob/develop/resources/execution_script_tags.json)
    * Exemplo: [scheduler.conf](./confs/scheduler/clean/scheduler.conf)

### Clone o repositório saps-quality-assurance
```
git clone -b https://github.com/ufcg-lsd/saps-quality-assurance ~/saps-quality-assurance
cd ~/saps-quality-assurance
```

### Execute os testes
* Comando: ```sudo bash bin start-systemtest <admin_email> <admin_password> <dispatcher_ip_addrres> <submission_rest_server_port>```
* Exemplo: ```sudo bash bin start-systemtest dispatcher_admin_email dispatcher_admin_password 127.0.0.1 8091```

-------------------------------------------------------------------
## [Crontab]
* catalog -> crontab do script de sumarização
  ```
  * * 1,15 * * sudo bash /home/ubuntu/saps-catalog/scripts/fetch_landsat_data.sh
  0 0 * * * bash /home/ubuntu/saps-catalog/scripts/build_tasks_overview.sh
  ```

* archiver -> crontab do script de contagem-dirs-arquivados
  ```
  * * */1 * * bash /home/ubuntu/saps-archiver/scripts/build_archiver_overview.sh
  ```

* dispatcher -> crontab do script de acessos + scripts de sumarização_manel
  ```
  59 23 * * * sudo bash /home/ubuntu/saps-dispatcher/scripts/login_counter.sh
  0 0 * * * sudo /bin/bash ~/saps-dispatcher/stats/stats_archived.sh > ~/saps-dispatcher/scripts/summary.csv 
  0 0 * * * sudo /bin/bash ~/saps-dispatcher/stats/logins_accumulator.sh >> ~/saps-dispatcher/scripts/summary.csv
  0 0 * * * sudo python3 ~/saps-dispatcher/stats/stats_tasks_raw_data.py
  ```

* arrebol -> crontab do script de limpeza do banco de dados
  ```
  0 0 * * *   sudo bash /home/ubuntu/arrebol/bin/db_cleaner.sh
  ```

* workers -> crontab dos containers não finalizados
  ```
  0 0 * * *  sudo docker ps -aq | sudo xargs docker stop | sudo xargs docker rm
  ```

-------------------------------------------------------------------
## [Logrotate]
* [dispatcher](./confs/dispatcher/logrotate.conf) 
* [scheduler](./confs/scheduler/logrotate.conf)
* [archiver](./confs/archiver/logrotate.conf)
* [arrebol](./confs/arrebol/logrotate.conf)
