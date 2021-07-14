# Monitoramento por ODBC Banco de dados Oracle

## Requisitos

Zabbix-server 4.4

O monitoramento ODBC corresponde ao tipo de item do monitor de banco de dados no frontend do Zabbix.
ODBC é uma API de middleware da linguagem de programação C para acessar sistemas de gerenciamento de banco de dados (DBMS). O conceito ODBC foi desenvolvido pela Microsoft e posteriormente transportado para outras plataformas.
O Zabbix pode consultar qualquer banco de dados, que seja compatível com ODBC. Para fazer isso, o Zabbix não se conecta diretamente aos bancos de dados, mas usa a interface ODBC e os drivers configurados no ODBC. Essa função permite o monitoramento mais eficiente de diferentes bancos de dados para diversos fins - por exemplo, verificar filas de banco de dados específicas, estatísticas de uso e assim por diante.

# Como Instalar 

## 1 - Instalar unixODBC 
```
yum -y install unixODBC unixODBC-devel
```

## 2 - Instalar os drivers basic e odbc necessários no proxy ou server 
```
yum install https://download.oracle.com/otn_software/linux/instantclient/185000/oracle-instantclient18.5-basic-18.5.0.0.0-3.x86_64.rpm
yum install https://download.oracle.com/otn_software/linux/instantclient/185000/oracle-instantclient18.5-odbc-18.5.0.0.0-3.x86_64.rpm
```

## 3 – Ajuste das variáveis de ambiente
```
echo /usr/lib/oracle/18.5/client64/lib/ >> /etc/ld.so.conf.d/oracle.conf
ldconfig
vim /etc/profile.d/oracle.sh
export LD_LIBRARY_PATH=/usr/lib/oracle/18.5/client64/lib:$LD_LIBRARY_PATH
export PATH=/usr/lib/oracle/18.5/client64/bin:$PATH
export TNS_ADMIN=/etc/oracle
source /etc/profile.d/oracle.sh
```
## 4 - Configurar o ODBC existe dois arquivos: **odbcinst.ini** para setar o driver utilizado e **obdc.ini** para setar a conexão ao banco
```
vim /etc/odbcinst.ini
[oracle]
Driver = /usr/lib/oracle/18.5/client64/lib/libsqora.so.18.1
FileUsage=1
Driver Logging=1
```
## Exemplo de arquivo odbc.ini 
Em **SERVERNAME** o nome colocado é alias que será colocado no tnsnames.ora
```
vim /etc/obdc.ini
[db01]
Driver = oracle
DSN = db01
ServerName = TOTPD
UserID = system
Password = Mandic123
[db02]
Driver = oracle
DSN = db02
ServerName = MXMPD
UserID = system
Password = Mandic123
```
## 5 – Crie diretório que será utilizado pelo arquivo tnsnames.ora 
```
mkdir /etc/oracle
```
## Abaixo exemplo do arquivo podendo inserir cada conexão conforme abaixo
```
vim /etc/oracle/tnsnames.ora

###Álias que teve seguir conforme colocado no arquivo odbc.ini
TOTPD =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 172.31.82.81)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = TOTPD_SVC)
  )
)

MXMPD =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 172.31.82.81)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = MXMPD_SVC)
  )
)
~
```
## 6 - Configure o servidor Zabbix ou proxy Zabbix para o uso Oracle ENV. Edite ou adicione um novo arquivo

```
vim /etc/sysconfig/zabbix-server
ORACLE_HOME=/usr/lib/oracle/18.5/client64
LD_LIBRARY_PATH=/usr/lib/oracle/18.5/client64/lib:$LD_LIBRARY_PATH
PATH=/usr/lib/oracle/18.5/client64/bin:$PATH
TNS_ADMIN=/etc/oracle

export ORACLE_HOME
export LD_LIBRARY_PATH
export TNS_ADMIN
export PATH
```
- Para verificar o PID do zabbix  
```
ps -axu | grep zabbix_server 
```
- Para verificar as variáveis de ambiente
```
strings -a /proc/**PID-zabbix-server**/environ 
```
Restart Zabbix server or Zabbix proxy

## 7 – Para testar a comunicação utilize o comando abaixo
```
isql -v db01 – DSN configurada no arquivo odbc.ini
 
```
## No front import o Template_ODBC_ORACLE e insira as macros no host
- {$ORACLE.USER} 
- {$ORACLE.PASSWORD}
- {$ORACLE.DSN}

## Alguns selects para testes
```
SELECT TABLESPACE_NAME FROM DBA_TABLESPACES;
select * from dba_tablespace_usage_metrics where tablespace_name in ('TBS1','TBS2') order by 1;
select file_id, tablespace_name, round(bytes/1024/1024) size_mb, blocks, autoextensible, increment_by, round(maxbytes/1024/1024) max_mb from dba_data_files where tablespace_name in ('TBS1') order by 1;
```
## Nota! Certifique-se de que o ODBC se conecta ao Oracle com o parâmetro de sessão NLS_NUMERIC_CHARACTERS = '.,' É importante para exibir números flutuantes no Zabbix corretamente senão o item fica como não suportado

Para verificar como está setado:
```
select * from v$nls_parameters;
```
Caso o parâmetro esteja dessa maneira **NLS_NUMERIC_CHARACTERS = ',.'**, configure no pre- processamento do item a seguinte expressão regular

```
(|[0-9]+),([0-9]+) - output \1.\2
```

## License
GPLV3
