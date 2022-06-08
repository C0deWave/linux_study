## 정규식을 통한 로그인 접근 아이피 추출하기

---

### **1. 로그 데이터 확인**

로그데이터에서 특정 데이터를 추출해 달라는 요청을 받았다.
grep 및 정규식을 통해서 해당 업무를 진행해 본다.
이런 업무를 할때는 항상 로그를 백업하고 백업한 로그에서 작업을 진행한다.

개인적으로 사용하는 odroid의 로그를 조사해 보자.
odroid는 우분투 계열이기 때문에 접속 로그를 auth.log에 저장한다.

~~~
root@odroid:/var/log# tail ./auth.log
Jun  9 00:55:01 odroid CRON[7015]: pam_unix(cron:session): session closed for user root
Jun  9 00:56:53 odroid sshd[8905]: Accepted password for odroid from 192.168.219.31 port 56798 ssh2
Jun  9 00:56:53 odroid sshd[8905]: pam_unix(sshd:session): session opened for user odroid by (uid=0)
Jun  9 00:56:54 odroid systemd-logind[740]: New session 27634 of user odroid.
Jun  9 00:56:54 odroid systemd: pam_unix(systemd-user:session): session opened for user odroid by (uid=0)
Jun  9 00:57:21 odroid dbus-daemon[718]: [system] Failed to activate service 'org.bluez': timed out (service_start_timeout=25000ms)
Jun  9 00:57:31 odroid sudo:   odroid : TTY=pts/0 ; PWD=/home/odroid ; USER=root ; COMMAND=/bin/su -
Jun  9 00:57:31 odroid sudo: pam_unix(sudo:session): session opened for user root by odroid(uid=0)
Jun  9 00:57:31 odroid su: (to root) odroid on pts/0
Jun  9 00:57:31 odroid su: pam_unix(su-l:session): session opened for user root by odroid(uid=0)
~~~

실제로 작업을 할 때는 로그를 백업하고 작업을 진행하자.

---

### **2. 로그인 기록을 추출한다.**

여기서 로그인에 성공한 로그를 추출해서 어느 아이피에서 접근 기록이 있었는지 확인해본다.
먼저 로그인 성공 로그를 추출해 낸다.
로그인 로그는 아래와 같다.

Accepted 라는 문자가 붙는 것으로 로그인에 성공한 로그를 추출할 수 있다.

~~~
### 예시
Jun  6 03:26:52 odroid sshd[26813]: Accepted password for odroid from 192.168.219.31 port 49293 ssh2
~~~

일반적을 grep을 이용해서 문자열을 추출하지만 압축된 gz 파일도 검사를 수행하기 위해서 zgrep를 이용한다.
-i 옵션은 대소문자 구분을 하지 않는다는 의미이다.
이것을 리다이렉트 시켜서 파일로 저장한다.

~~~
root@odroid:/var/log# zgrep -i accept ./auth.log*  > accept.log
root@odroid:/var/log# tail accept.log
./auth.log:Jun  6 03:26:52 odroid sshd[26813]: Accepted password for odroid from 192.168.219.31 port 49293 ssh2
./auth.log:Jun  9 00:56:53 odroid sshd[8905]: Accepted password for odroid from 192.168.219.31 port 56798 ssh2
./auth.log.1:May 29 22:53:13 odroid sshd[14763]: Accepted password for odroid from 192.168.219.31 port 56732 ssh2
~~~

---

### **3. 로그인 아이피를 정규식을 통해서 추출한다.**

지금은 192.168.219.31에서 3번 접근한 것을 알 수 있지만 개수가 수백 수천개일 경우 이를 일일이 셀 수 없기 때문에 정규식을 이용한 중복 제거를 한다.

정규식은 grep -o 명령어를 통해서 진행한다.
-o 옵션은 해당 정규식 만 추출을 한다.
-E 확장 정규식으로 검색을 진행한다.
ip 정규식을 적어 추출해보자.

~~~
root@odroid:/var/log# grep -Eo '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' ./accept.log
192.168.219.31
192.168.219.31
192.168.219.31
~~~

---

### **4. uniq 명령어를 통해서 개수를 센다.**

이를 uniq를 통해서 개수를 센다.
uniq명령어는 윗줄 아랫줄만 확인하기 때문에 그전에 정렬을 진행한다.

~~~
root@odroid:/var/log# grep -Eo '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' ./accept.log  | sort | uniq -c
      3 192.168.219.31
~~~

이것으로 219.31 아이피에서 3번의 접근을 한 것을 알 수 있다.
이를 응용하면 유저별로 몇번을 접근했는지 등을 알 수 있다.