Các chức năng SSH Proxy

    Log được thông tin SSH của nhân viên như thời gian login, user login, IP máy chủ …
    Log được lệnh khi gõ qua SSH, output của lệnh.
    Người quản trị có thể kiểm rta log trong trường hợp cần thiết.

Mô hình hoạt động SSH Proxy

Ví dụ đơn vị bạn là một hệ thống lớn, mỗi sáng nhân viên đi làm sẽ sử dụng thiết bị cá nhân với user được cấp sẽ truy cập vào hệ thống để làm việc. Do đó toàn bộ nhân viên phải SSH qua SSH proxy mới vào được vào bên trong và các biện pháp log thông tin trên server SSH proxy cụ thể là:

    Những server/VPS cần log phải thực hiện giới hạn IP cho phép login, và chỉ được phép login từ IP của SSH Proxy
    Trên SSH Proxy cấp cho nhân viên user để SSH vào server Proxy trước khi SSH vào server khác
    Trên Server SSH proxy triển khai các biện pháp log thông tin SSH, lệnh và output…

Ví dụ:

    Server SSH Proxy có IP là: 123.123.123.1
    Server cần SSH vào làm việc: 123.123.123.2

Các bước cài đặt server SSH Proxy
Bước 1: Cấu hình SSH chỉ cho phép chứng thực bằng SSH key

Để cấu hình cho phép chứng thực bằng SSH Key và không sử dụng passwd root bạn cần mở file sshd_config và sửa lại như sau.

vi  /etc/ssh/sshd_config
    

cấu hình SSH Proxy

Sau khi đã sửa xong bạn khởi động lại dịch vụ sshd bằng lệnh sau

systemctl restart sshd
    

Bước 2: Cấu hình Firewall

Bạn hãy cài đặt CSF firewall lên server SSH Proxy. Nếu bạn chưa thực hiện cài đặt hãy tham khảo tải liệu sau nhé.

    Cài đặt và cấu hình CSF (Config Server Firewall) trên CentOS 7
    Giải thích và sử dụng CSF (ConfigServer & Firewall)

Sau đó bạn mở file /etc/csf/csf.conf và sửa lại như sau.

vi /etc/csf/csf.conf
    


# Allow incoming TCP ports
TCP_IN = ""

# Allow outgoing
TCP por ts TCP_OUT = "53"

# Allow incoming UDP ports
UDP_IN = "53"

# Allow outgoing UDP ports
UDP_OUT = "53"
    

Tiếp theo bạn hãy cấu hình chỉ cho phép người dùng SSH có IP là IP VPN tại /etc/csf/csf.allow

tcp | in | d=22 | s=123.123.123.123 # VPN IP (Thay IP này bằng IP VPN của bạn)
    

Lưu ý: Bạn cũng có thể tạo một file file-name.allow sau đó include vào /etc/csf/csf.allow

Để cấu hình để không tracking các IP VPN này tại file /etc/csf/csf.ignore

Lưu ý: Bạn cũng có thể tạo một file file-name.ignore sau đó include vào /etc/csf/csf.ignore
Bước 3: Cài đặt các script để log SSH session.
1. log-session

    Bạn hãy tạo một file /usr/local/sbin/log-session và dán nội dung như sau vào nhé.


vi /usr/local/sbin/log-session
    

    Nội dung file log-session

!/bin/sh
 #
 NOW=date +%Y -%m -%d.%H%M%S
 IP=echo $SSH_CLIENT | sed 's/ .*//'
 USER=whoami
 LOGDIR="/log -session"
 LOGFILE=$LOGDIR/$USER/$USER.log
 echo ======================================== >> $LOGFILE echo Starting interactive shell session by user $USER - IP: $IP - $NOW >>
 $LOGFILE
 echo ======================================== >> $LOGFILE exec script -a -f -q

    Phân quyền cho file /usr/local/sbin/log-session


chmod 755 /usr/local/sbin/log-session
    

Script này khi chạy sẽ lấy thông tin về user đang ssh vào qua lệnh whoami và lấy thời điểm ssh bằng lệnh date. Lệnh script sẽ copy một bản SSH terminate vào file log.

Script log-session này được gọi đầu tiên khi người dùng SSH vào thông qua chỉ định command tại file public key ($HOME/.ssh/.authorized_keys)
2. Script adduser

Tạo script vi /root/adduser.sh script này dùng để tạo user cho thành viên kỹ thuật, nội dung như sau.

vi /root/adduser.sh
    

    Nội dung adduser.sh

/bin/bash
 Script add user to log ssh session
 LOG_DIR="/log-session"
 PASSWORD=< /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c12;echo
 USER=$1 KEY_TYPE=$2 SSH_KEY=$3
 if [ -z "$USER" ] || [ -z "$KEY_TYPE" ] || [ -z "$SSH_KEY" ]; then
 echo Error. User or SSH key is not empty exit 0
 fi
 useradd $USER -p $PASSWORD && echo Success create user $USER || echo Fail create user $USER
 mkdir -p /log-session/$USER
 touch $LOG_DIR/$USER/$USER.log && chmod 200 $LOG_DIR/$USER/$USER.log && chown $USER. $LOG_DIR/$USER/$USER.log && chattr +a $LOG_DIR/$USER/$USER.log && echo Success create and change permission log file || echo Fail create and change permission log file
 mkdir /home/$USER/.ssh && chown $USER. /home/$USER/.ssh && chmod 700 /home/$USER/.ssh && echo Success create /home/$USER/.ssh directory || echo Fail create /home/$USER/.ssh directory
 echo command=\"/usr/local/sbin/log-session\" $KEY_TYPE $SSH_KEY > /home/$USER/.ssh/authorized_keys && chmod 400 /home/$USER/.ssh/authorized_keys && chown -R $USER. /home/$USER/.ssh && chattr +i /home/$USER/.ssh/authorized_keys && echo Success create /home/$USER/.ssh/authorized_keys file || echo Fail create /home/$USER/.ssh/authorized_keys file
 echo Done!

Script này được dùng để thêm user khi cần cấp user cho người dùng proxy, đồng thời thực hiện thêm public key, tạo file log, phân quyền cho file log để ngăn người dùng xem, sửa file log. Cụ thể là:

    Đọc thông tin user, public key nhập vào bằng lệnh read
    Thực hiện thêm user thông tin qua useradd
    Tạo thư mục chứa log file cho từng user qua lệnh mkdir -p /log-session/$USER
    Tạo file log cho user đó thông qua lệnh touch

Phân quyền:

    200 với logfile
    thay đổi owner user/group là chính user đó
    chattr +a file log để user chỉ thêm nội dung vào mà không có quyền chỉnh sửa hay xóa nội dung
    Tạo thư mục /home/$USER/.ssh, phân quyền 700 cho thư mục này.
    Tạo file /home/$USER/.ssh/authorized_keys, chown user/group là root.$USER với permission là 750. Phân quyền như trên sẽ giúp user chỉ đọc được các file trong thư mục /home/$USER/.ssh và không thể tạo thêm được file trong thư mục này. Thêm nội dung public key của user vào với định dạng : command = “/usr/local/sbin/log-session” ssh-rsa AAAQE…… user@quandt@azdigi.vn
    chattr +i /home/$USER/.ssh/authorized_keys để user không chỉnh sửa được file này

3. Tạo script log_rotate.sh

    Tạo script rotate log tại /root/log_rotate.sh, nội dung như sau


vi /root/log_rotate.sh
    

!/bin/bash
 TIMESTAMP=date +%d-%m-%Y
 LOGDIR=/log-session/
 MAX_SIZE=100M
 find $LOGDIR -name '*.log' -type f -size +$MAX_SIZE | while read LOGFILE
 do
 done

    Phân quyền cho script /root/log_rotate.sh


chmod 755 /root/log_rotate.sh
    

    Thêm cronjob tự động rotate log

Script dùng để rotate file log, các file log có dung lượng lớn hơn biến $MAX_SIZE sẽ được rotate

0 2 * * * /root/log_rotate.sh > /dev/null
    

Bước 4: Cài đặt trên server/VPS cần log SSH session

Trên server/VPS cần log SSH, cần thực hiện việc giới hạn login SSH, chỉ cho phép IP của SSH Proxy được phép đi vào. Thực hiện như sau :

Trên server/VPS cần log SSH, vào /etc/ssh/sshd_config, thêm giá trị sau vào cuối file.

vi /etc/ssh/sshd_config
    


Match Address  
PasswordAuthentication no 
PermitRootLogin yes 
    

Bước 5: Tạo user

Để tạo user bạn hãy SSH vào con proxy để làm việc nhé và để tạo user bạn thực hiện với cú pháp lệnh sau:

Trong đó

    username: Là user của người dùng
    ssh-dss AAAAB3…: Chính là public_key của người dùng đó


./root/adduser.sh username ssh-dss AAAAB3...
    

Sau khi tạo hoàn tất, người dùng có thể sử dụng privakey và user để ssh vào Proxy. Và tải lên Private key để ssh đến những server khác
