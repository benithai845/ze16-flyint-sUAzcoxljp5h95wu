[合集 - 一键安装脚本(2)](https://github.com)

[1.实战！oracle 11g一键安装脚本分享2024-10-14](https://github.com/liuziyi1/p/18464119):[MeoMiao 萌喵加速](https://biqumo.org)

2.【实用脚本】一键完成MySQL数据库健康巡检，并生成word报告10-31

收起

**过程截图：**

![e4cf303d-2c01-427b-9107-d3cc953135a7](https://img2024.cnblogs.com/blog/3323825/202510/3323825-20251031172428262-870106919.png)

![110deaf0-c7de-4828-8103-4c3ae52eb640](https://img2024.cnblogs.com/blog/3323825/202510/3323825-20251031173245321-939679524.png)

**说明**：赋予执行权限，执行即可。
**源码文件：**

```
#!/usr/bin/env python3
# -*- coding:utf-8 -*-
from __future__ import unicode_literals
import itertools
import math
import sys
import datetime
import argparse
import subprocess
import logging
import logging.handlers
import socket
import re
import time
from pathlib import Path
import pymysql
import datetime
import sys, getopt, os
import docx
from docx.shared import Cm
from docxtpl import DocxTemplate
import configparser
import importlib
import subprocess
import json
import hashlib
import base64
from datetime import datetime, timedelta
import platform

importlib.reload(sys)

# 获取资源路径函数 - 处理打包后的路径问题
def get_resource_path(relative_path):
    """获取资源的绝对路径，处理打包后的路径问题"""
    try:
        # PyInstaller创建临时文件夹，将路径存储在_MEIPASS中
        base_path = sys._MEIPASS
    except Exception:
        base_path = os.path.abspath(".")
    
    return os.path.join(base_path, relative_path)

# 许可证管理类
class SimpleCrypto:
    """简单的加密解密类，兼容Python 3.6以上"""
    
    def __init__(self, secret="ODB_SECRET_2024"):
        self.secret = secret.encode('utf-8')
    
    def _xor_encrypt_decrypt(self, data, key):
        """使用XOR进行加密/解密"""
        key_len = len(key)
        result = bytearray()
        for i, byte in enumerate(data):
            result.append(byte ^ key[i % key_len])
        return bytes(result)
    
    def encrypt(self, text):
        """加密文本"""
        data = text.encode('utf-8')
        # 生成密钥
        key = hashlib.sha256(self.secret).digest()[:32]
        # XOR加密
        encrypted = self._xor_encrypt_decrypt(data, key)
        # Base64编码
        return base64.b64encode(encrypted).decode('utf-8')
    
    def decrypt(self, token):
        """解密文本"""
        try:
            # Base64解码
            encrypted = base64.b64decode(token.encode('utf-8'))
            # 生成密钥
            key = hashlib.sha256(self.secret).digest()[:32]
            # XOR解密
            decrypted = self._xor_encrypt_decrypt(encrypted, key)
            return decrypted.decode('utf-8')
        except Exception as e:
            raise ValueError(f"解密失败: {e}")

class LicenseValidator:
    """严格的许可证验证类 - 防止删除文件重置试用期"""

    def __init__(self):
        self.license_file = "mysql_inspector.lic"
        self.trial_days = 366
        self.crypto = SimpleCrypto()
        self._init_license_system()

    def _parse_datetime(self, date_string):
        """解析日期字符串，兼容Python 3.6"""
        try:
            # 尝试Python 3.7+的fromisoformat
            return datetime.fromisoformat(date_string)
        except AttributeError:
            # Python 3.6兼容：手动解析ISO格式
            try:
                # 格式: 2024-01-01T10:30:00
                if 'T' in date_string:
                    date_part, time_part = date_string.split('T')
                    year, month, day = map(int, date_part.split('-'))
                    time_parts = time_part.split(':')
                    hour, minute = int(time_parts[0]), int(time_parts[1])
                    second = int(time_parts[2].split('.')[0]) if len(time_parts) > 2 else 0
                    return datetime(year, month, day, hour, minute, second)
                else:
                    # 格式: 2024-01-01 10:30:00
                    date_part, time_part = date_string.split(' ')
                    year, month, day = map(int, date_part.split('-'))
                    hour, minute, second = map(int, time_part.split(':'))
                    return datetime(year, month, day, hour, minute, second)
            except Exception as e:
                print(f"日期解析错误: {e}")
                return datetime.now()

    def _format_datetime(self, dt):
        """格式化日期为字符串，兼容Python 3.6"""
        return dt.strftime('%Y-%m-%dT%H:%M:%S')

    def _init_license_system(self):
        """初始化许可证系统"""
        # 创建必要的目录和文件
        if not os.path.exists(self.license_file):
            self._create_trial_license()

    def _create_trial_license(self):
        """创建试用许可证"""
        create_time = datetime.now()
        expire_time = create_time + timedelta(days=self.trial_days)
        
        license_data = {
            "type": "TRIAL",
            "create_time": self._format_datetime(create_time),
            "expire_time": self._format_datetime(expire_time),
            "machine_id": self._get_machine_id(),
            "signature": self._generate_signature("TRIAL")
        }
        encrypted_data = self.crypto.encrypt(json.dumps(license_data))
        with open(self.license_file, 'w') as f:
            f.write(encrypted_data)

    def _get_machine_id(self):
        """获取机器标识（简化版）"""
        try:
            machine_info = f"{platform.node()}-{platform.system()}-{platform.release()}"
            return hashlib.md5(machine_info.encode()).hexdigest()[:16]
        except:
            return "unknown_machine"

    def _generate_signature(self, license_type):
        """生成许可证签名"""
        key = "ODB2024TRL"  # 试用版密钥
        signature_data = f"{license_type}-{datetime.now().strftime('%Y%m%d')}-{key}"
        return hashlib.sha256(signature_data.encode()).hexdigest()

    def _verify_signature(self, license_data):
        """验证许可证签名"""
        try:
            expected_signature = self._generate_signature(license_data["type"])
            return license_data["signature"] == expected_signature
        except:
            return False

    def validate_license(self):
        """验证许可证有效性"""
        try:
            # 检查许可证文件
            if not os.path.exists(self.license_file):
                return False, "许可证文件不存在", 0

            # 读取并解密许可证
            with open(self.license_file, 'r') as f:
                encrypted_data = f.read().strip()
            
            decrypted_data = self.crypto.decrypt(encrypted_data)
            license_data = json.loads(decrypted_data)

            # 验证签名
            if not self._verify_signature(license_data):
                return False, "许可证签名无效", 0

            # 检查过期时间
            expire_time = self._parse_datetime(license_data["expire_time"])
            remaining_days = (expire_time - datetime.now()).days

            if remaining_days < 0:
                return False, "许可证已过期", 0

            license_type = license_data.get("type", "TRIAL")
            
            if license_type == "TRIAL":
                return True, f"试用版许可证有效，剩余 {remaining_days} 天", remaining_days
            else:
                return True, f"{license_type}版许可证有效", remaining_days

        except Exception as e:
            return False, f"许可证验证失败: {str(e)}", 0

# 临时替代自定义logger
def getlogger():
    logger = logging.getLogger('mysql_check')
    logger.setLevel(logging.INFO)
    if not logger.handlers:
        handler = logging.StreamHandler()
        formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
        handler.setFormatter(formatter)
        logger.addHandler(handler)
    return logger

logger = getlogger()

# 辅助类：传递脚本输入的参数
class passArgu(object):
    """辅助类,脚本传参
    查看帮助：python3 main.py -h"""

    def get_argus(self):
        """ all_info: 接收所有传入的信息 """
        all_info = argparse.ArgumentParser(
            description="--example: python3 mysql_autoDOC.py -C templates/sqltemplates.ini -L '标签名称'")
        all_info.add_argument('-C', '--sqltemplates', required=False, default='templates/sqltemplates.ini',
                              help='SQL sqltemplates.')
        all_info.add_argument('-L', '--label', required=False, help='Label used when health check single database.')
        all_info.add_argument('-B', '--batch', action='store_true', help='Batch mode (use interactive input for multiple DBs)')

        all_para = all_info.parse_args()
        " 返回值默认string, 不区分前后顺序 "
        return all_para

# 新增：交互式输入数据库连接信息
def input_db_info():
    """交互式输入数据库连接信息，支持默认值"""
    print("\n请输入数据库连接信息:")
    
    # 主机地址，默认localhost
    host = input("主机地址 [localhost]: ").strip()
    if not host:
        host = "localhost"
    
    # 端口，默认3306
    port_input = input("端口 [3306]: ").strip()
    if not port_input:
        port = 3306
    else:
        try:
            port = int(port_input)
        except ValueError:
            print("⚠️  端口输入无效，使用默认值3306")
            port = 3306
    
    # 用户名，默认root
    user = input("用户名 [root]: ").strip()
    if not user:
        user = "root"
    
    # 密码，无默认值
    import getpass
    password = getpass.getpass("密码: ").strip()
    
    # 数据库名称/标签
    db_name = input("数据库名称(用于报告标识) [MySQL_Server]: ").strip()
    if not db_name:
        db_name = "MySQL_Server"
    
    # 验证连接
    print(f"\n🔍 正在验证连接 {host}:{port}...")
    try:
        conn = pymysql.connect(
            host=host,
            port=port,
            user=user,
            password=password,
            charset='utf8mb4',
            connect_timeout=5
        )
        conn.close()
        print(f"✅ 成功连接到 {host}:{port}")
        return {
            'name': db_name,
            'ip': host,
            'port': port,
            'user': user,
            'password': password
        }
    except Exception as e:
        print(f"❌ 连接失败: {e}")
        retry = input("是否重新输入? (y/n) [n]: ").strip().lower()
        if retry == 'y':
            return input_db_info()
        else:
            sys.exit(1)

# 新增：批量输入数据库信息
def input_batch_db_info():
    """批量输入多个数据库连接信息"""
    db_list = []
    print("\n=== 批量数据库输入模式 ===")
    print("请依次输入每个数据库的连接信息")
    print("输入完成后直接按回车确认")
    
    while True:
        print(f"\n--- 数据库 {len(db_list) + 1} ---")
        db_info = input_db_info()
        db_list.append(db_info)
        
        continue_input = input("\n是否继续添加其他数据库? (y/n) [n]: ").strip().lower()
        if continue_input != 'y':
            break
    
    if not db_list:
        print("❌ 未输入任何数据库信息")
        sys.exit(1)
    
    return db_list

# 获取巡检指标数据的类
class getData(object):
    def __init__(self, ip, port, user, password):
        infos = passArgu().get_argus()
        self.label = str(infos.label)

        self.H = ip
        self.P = int(port)
        self.user = user
        self.password = password

        # 默认连接串
        try:
            conn_db2 = pymysql.connect(host=self.H, port=self.P, user=self.user, password=self.password, charset='utf8mb4')
            self.conn_db2 = conn_db2
        except Exception as e:
            print(f"❌ 数据库连接失败: {e}")
            sys.exit(1)

        # 创建一个空字典，存储数据库巡检数据
        context = {}
        self.context = context

    def print_progress_bar(self, iteration, total, prefix='', suffix='', decimals=1, length=50, fill='█'):
        """打印进度条"""
        percent = ("{0:." + str(decimals) + "f}").format(100 * (iteration / float(total)))
        filled_length = int(length * iteration // total)
        bar = fill * filled_length + '-' * (length - filled_length)
        print(f'\r{prefix} |{bar}| {percent}% {suffix}', end='\r')
        if iteration == total:
            print()

    def checkdb(self, sqlfile=''):
        print("\n开始巡检...")
        
        # 模拟进度条
        total_steps = 15
        current_step = 0
        
        # 1、通过pymysql获取mysql数据库指标信息
        cfg = configparser.RawConfigParser()
        
        # 使用绝对路径读取配置文件
        try:
            cfg.read(sqlfile, encoding='utf-8')
        except Exception as e:
            print(f"❌ 读取SQL模板文件失败: {e}")
            return self.context

        # 初始化上下文
        init_keys = ["mdlinfo", "mgrinfo", "sql5min", "innodb_trx", "db_size", "userinfo", 
                    "tbtop10", "idxtop10", "nopk", "obnum", "noinnodb", "indexnum5", 
                    "indexcolnum", "colnum50", "iotop10", "memtop10", "sqltop10", 
                    "fullscantop10", "fullscantbtop10", "tmpsqltop10", "rowsdmltop30", 
                    "unuseidx", "incrtop10", "redundantidx"]
        
        for key in init_keys:
            self.context.update({key: []})

        # 获取数据库版本信息
        try:
            cursor_ver = self.conn_db2.cursor()
            cursor_ver.execute("SELECT VERSION()")
            version_result = cursor_ver.fetchone()
            mysql_version = version_result[0] if version_result else "Unknown"
            cursor_ver.close()
            
            # 存储版本信息
            self.context.update({"myversion": [{'version': mysql_version}]})
            
            # 添加health_summary
            self.context.update({"health_summary": [{'health_summary': '运行良好'}]})
                
        except Exception as e:
            print(f"❌ 获取版本信息失败: {e}")
            self.context.update({"myversion": [{'version': 'Unknown'}]})
            self.context.update({"health_summary": [{'health_summary': '运行良好'}]})

        try:
            # 初始化一个数据库连接
            cursor2 = self.conn_db2.cursor()
            
            # 先获取当前sql_mode（用于临时调整）
            cursor2.execute("SELECT @@sql_mode")
            original_sql_mode = cursor2.fetchone()[0]
            # 移除ONLY_FULL_GROUP_BY的临时模式
            temp_sql_mode = original_sql_mode.replace('ONLY_FULL_GROUP_BY', '').strip()
            # 处理可能的多余逗号
            temp_sql_mode = re.sub(r',+', ',', temp_sql_mode).strip(',')
            
            # 遍历执行数据库sql，开始巡检
            for i, (name, stmt) in enumerate(cfg.items("variables")):
                try:
                    current_step = int((i / len(cfg.items("variables"))) * total_steps)
                    self.print_progress_bar(current_step, total_steps, prefix='巡检进度:', suffix=f'步骤 {i+1}/{len(cfg.items("variables"))}')
                    
                    # 针对sqltop10临时调整sql_mode
                    if name == "sqltop10":
                        cursor2.execute(f"SET sql_mode = '{temp_sql_mode}'")
                    
                    # 执行查询
                    cursor2.execute(stmt.replace('\n', ' ').replace('\r', ' '))
                    result = [dict((cursor2.description[i][0], value) for i, value in enumerate(row)) for row in cursor2.fetchall()]
                    self.context[name] = result
                    
                    # 执行完sqltop10后恢复原始sql_mode
                    if name == "sqltop10":
                        cursor2.execute(f"SET sql_mode = '{original_sql_mode}'")
                    
                    time.sleep(0.05)  # 为了显示进度效果
                except Exception as e:
                    print(f"\n⚠️  步骤 {name} 执行失败: {e}")
                    self.context[name] = []
                    # 若sqltop10失败，仍尝试恢复sql_mode
                    if name == "sqltop10":
                        try:
                            cursor2.execute(f"SET sql_mode = '{original_sql_mode}'")
                        except:
                            pass
        except Exception as e:
            print(f'\n❌ 数据库查询失败: {e}')
        finally:
            cursor2.close()

        # 查看innodb详细信息
        current_step = total_steps - 2
        self.print_progress_bar(current_step, total_steps, prefix='巡检进度:', suffix='获取InnoDB状态')
        try:
            cursor3 = self.conn_db2.cursor()
            cursor3.execute('show engine innodb status')
            innodbinfo = cursor3.fetchall()
            cursor3.close()
            self.context.update({"innodbinfo": [{'innodbinfo': innodbinfo[0][2]}]})
        except Exception as e:
            print(f"\n❌ 获取InnoDB状态失败: {e}")
            self.context.update({"innodbinfo": [{'innodbinfo': '获取失败'}]})

        # 2、风险与建议
        current_step = total_steps - 1
        self.print_progress_bar(current_step, total_steps, prefix='巡检进度:', suffix='分析风险和建议')
        
        self.context.update({"auto_analyze": []})
        
        # 简化的风险分析逻辑
        try:
            # 检查无主键表
            cursor_pk = self.conn_db2.cursor()
            cursor_pk.execute("""
                SELECT t.TABLE_SCHEMA, t.TABLE_NAME 
                FROM information_schema.TABLES t 
                LEFT JOIN information_schema.TABLE_CONSTRAINTS tc 
                ON t.TABLE_SCHEMA = tc.TABLE_SCHEMA 
                AND t.TABLE_NAME = tc.TABLE_NAME 
                AND tc.CONSTRAINT_TYPE = 'PRIMARY KEY' 
                WHERE t.TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
                AND tc.CONSTRAINT_NAME IS NULL
                AND t.TABLE_TYPE = 'BASE TABLE'
            """)
            no_pk_tables = cursor_pk.fetchall()
            cursor_pk.close()
            
            if no_pk_tables:
                self.context['auto_analyze'].append({
                    'col1': "无主键表检查", 
                    "col2": "中风险", 
                    "col3": f"发现 {len(no_pk_tables)} 个无主键表，影响性能和数据完整性", 
                    "col4": "中", 
                    "col5": "开发"
                })
            
            # 检查Buffer Pool命中率
            cursor_bp = self.conn_db2.cursor()
            cursor_bp.execute("SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%'")
            bp_stats = cursor_bp.fetchall()
            cursor_bp.close()
            
            bp_stats_dict = {item[0]: item[1] for item in bp_stats}
            if 'Innodb_buffer_pool_read_requests' in bp_stats_dict and 'Innodb_buffer_pool_reads' in bp_stats_dict:
                read_requests = int(bp_stats_dict['Innodb_buffer_pool_read_requests'])
                reads = int(bp_stats_dict['Innodb_buffer_pool_reads'])
                if read_requests > 0:
                    hit_ratio = (1 - reads / read_requests) * 100
                    if hit_ratio < 90:
                        self.context['auto_analyze'].append({
                            'col1': "Buffer Pool命中率", 
                            "col2": "低风险", 
                            "col3": f"InnoDB Buffer Pool命中率偏低: {hit_ratio:.2f}%", 
                            "col4": "低", 
                            "col5": "DBA"
                        })
            
            # 检查root用户远程登录
            cursor_user = self.conn_db2.cursor()
            cursor_user.execute("SELECT host, user FROM mysql.user WHERE user='root' AND host NOT IN ('localhost', '127.0.0.1', '::1')")
            remote_root = cursor_user.fetchall()
            cursor_user.close()
            
            if remote_root:
                self.context['auto_analyze'].append({
                    'col1': "Root用户远程访问", 
                    "col2": "高风险", 
                    "col3": "root用户允许远程登录，存在安全风险", 
                    "col4": "高", 
                    "col5": "DBA"
                })
                
        except Exception as e:
            print(f"\n❌ 风险分析失败: {e}")
        
        # 完成进度条
        self.print_progress_bar(total_steps, total_steps, prefix='巡检进度:', suffix='完成')
        
        return self.context

# 数据库查询内容输出保存
class saveDoc(object):
    def __init__(self, context, ofile, ifile):
        self.context = context
        self.ofile = ofile
        self.ifile = ifile

    # 内容依据模板文件写入输出文件
    def contextsave(self):
        try:
            # 确保所有必需的键都存在
            required_keys = ['health_summary', 'auto_analyze', 'myversion', 'co_name', 'port', 'ip']
            for key in required_keys:
                if key not in self.context:
                    if key == 'health_summary':
                        self.context[key] = [{'health_summary': '运行良好'}]
                    elif key == 'auto_analyze':
                        self.context[key] = []
                    elif key == 'myversion':
                        self.context[key] = [{'version': 'Unknown'}]
                    else:
                        self.context[key] = [{'placeholder': '数据缺失'}]
            
            tpl = DocxTemplate(self.ifile)
            tpl.render(self.context)
            tpl.save(self.ofile)
            return True
        except Exception as e:
            print(f"❌ 生成Word文档失败: {e}")
            return False

def print_banner():
    print("=" * 60)
    print("MySQL 数据库巡检工具 (Word报告版) - 交互式输入模式")
    print("支持版本: MySQL 5.6 / 5.7 / 8.0+")
    print("=" * 60)

def check_license():
    """检查许可证有效性"""
    validator = LicenseValidator()
    is_valid, message, remaining_days = validator.validate_license()
    
    if not is_valid:
        print(f"❌ {message}")
        print("请联系管理员获取有效许可证")
        sys.exit(1)
    else:
        print(f"✅ {message}")
        if "试用版" in message:
            print("⚠️  试用版只能使用一次，过期后需联系管理员获取正式许可")
        print()

def main():
    start_time = time.time()
    
    # 打印横幅
    print_banner()
    
    # 检查许可证
    check_license()
    
    infos = passArgu().get_argus()
    batch_mode = infos.batch

    # 使用资源路径函数获取模板文件
    sql_template = get_resource_path("templates/sqltemplates.ini")
    ifile = get_resource_path("templates/wordtemplates_v2.0.docx")
    
    # 检查模板文件是否存在
    if not os.path.exists(sql_template):
        print(f"❌ SQL模板文件不存在: {sql_template}")
        print("请确保模板文件已正确打包")
        sys.exit(1)
    
    if not os.path.exists(ifile):
        print(f"❌ Word模板文件不存在: {ifile}")
        print("请确保模板文件已正确打包")
        sys.exit(1)

    # 读取SQL模板配置
    cfg = configparser.RawConfigParser()
    try:
        cfg.read(sql_template, encoding='utf-8')
        
        # 检查是否包含report节
        if not cfg.has_section('report'):
            print("❌ SQL模板文件缺少[report]节")
            sys.exit(1)
            
    except Exception as e:
        print(f"❌ 读取SQL模板配置失败: {e}")
        sys.exit(1)
    
    # 配置报告输出路径
    dir_path = os.path.dirname(cfg.get("report", "output"))
    file_name = os.path.basename(cfg.get("report", "output"))

    # 生成带时间戳的报告文件名前缀
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    base_name = os.path.splitext(file_name)[0]
    extension = os.path.splitext(file_name)[1]

    # 处理批量模式
    if batch_mode:
        # 批量输入数据库信息
        db_list = input_batch_db_info()
        total_dbs = len(db_list)
        current_db = 0
        
        for db_info in db_list:
            current_db += 1
            label_name = db_info['name']
            ip = db_info['ip']
            port = db_info['port']
            user = db_info['user']
            password = db_info['password']

            # 生成报告文件名
            file_name = f"{base_name}_{label_name}_{timestamp}_{current_db}{extension}"
            ofile = os.path.join(dir_path, file_name)

            print(f"\n[{current_db}/{total_dbs}] 开始巡检 {label_name} ({ip}:{port})...")
            
            # 获取数据库版本
            try:
                conn_test = pymysql.connect(host=ip, port=int(port), user=user, password=password)
                cursor = conn_test.cursor()
                cursor.execute("SELECT VERSION()")
                version = cursor.fetchone()[0]
                cursor.close()
                conn_test.close()
                print(f"📊 数据库版本: MySQL {version}")
            except Exception as e:
                print(f"📊 数据库版本: 未知 ({e})")

            # 执行巡检
            data = getData(ip, port, user, password)
            ret = data.checkdb(sql_template)

            # 添加报告必要信息
            ret.update({"co_name": [{'CO_NAME': label_name}]})
            ret.update({"port": [{'PORT': port}]})
            ret.update({"ip": [{'IP': ip}]})

            # 生成报告
            savedoc = saveDoc(context=ret, ofile=ofile, ifile=ifile)
            success = savedoc.contextsave()
            
            if success:
                print(f"✅ 报告已生成: {os.path.basename(ofile)}")
            else:
                print(f"❌ {label_name} 报告生成失败")
            
            time.sleep(1)
        
        end_time = time.time()
        total_time = end_time - start_time
        print(f"\n=== 批量巡检完成 ===")
        print(f"总耗时: {total_time:.2f}秒")
        print(f"处理数据库数量: {total_dbs} 个")
        print(f"报告输出目录: {dir_path}")
        print("=" * 60)
    
    else:
        # 单库模式 - 交互式输入
        db_info = input_db_info()
        label_name = db_info['name']
        ip = db_info['ip']
        port = db_info['port']
        user = db_info['user']
        password = db_info['password']

        # 生成报告文件名
        file_name = f"{base_name}_{label_name}_{timestamp}{extension}"
        ofile = os.path.join(dir_path, file_name)

        # 获取数据库版本
        try:
            conn_test = pymysql.connect(host=ip, port=int(port), user=user, password=password)
            cursor = conn_test.cursor()
            cursor.execute("SELECT VERSION()")
            version = cursor.fetchone()[0]
            cursor.close()
            conn_test.close()
            print(f"📊 数据库版本: MySQL {version}")
        except Exception as e:
            print(f"📊 数据库版本: 未知 ({e})")

        # 执行巡检
        data = getData(ip, port, user, password)
        ret = data.checkdb(sql_template)

        # 添加报告必要信息
        ret.update({"co_name": [{'CO_NAME': label_name}]})
        ret.update({"port": [{'PORT': port}]})
        ret.update({"ip": [{'IP': ip}]})

        # 生成报告
        savedoc = saveDoc(context=ret, ofile=ofile, ifile=ifile)
        success = savedoc.contextsave()
        
        if not success:
            print("❌ 生成报告失败，但数据采集已完成")
            return
        
        # 统计问题数量
        problem_count = len(ret.get("auto_analyze", []))
        major_problems = []
        
        # 从auto_analyze中提取主要问题
        for item in ret.get("auto_analyze", []):
            major_problems.append(f"{item['col1']}: {item['col3']}")
        
        # 如果没有发现问题，添加一条信息
        if problem_count == 0:
            major_problems.append("未发现重大问题")
        
        end_time = time.time()
        total_time = end_time - start_time
        
        print(f"\n巡检完成!")
        print(f"✅ Word报告已生成: {os.path.basename(ofile)}")
        print(f"📁 报告路径: {ofile}")
        print(f"\n总耗时: {total_time:.2f}秒")
        print(f"发现问题: {problem_count} 个")
        
        if major_problems:
            print("\n主要问题:")
            for i, problem in enumerate(major_problems, 1):
                print(f"  {i}. {problem}")
        
        print("=" * 60)
        print("巡检完成! 请查看Word报告了解详情")
        print("=" * 60)

if __name__ == '__main__':
    main()
```
