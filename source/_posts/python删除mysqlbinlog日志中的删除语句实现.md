---
title: python删除mysqlbinlog日志中的删除语句实现
date: 2016-09-25 10:50:36
desc: 
tags: python
---

昨天写的mysqlbinlog处理数据的帖子, 最后有个问题, 就是数据备份出来的大小有好几G, 单纯的用编辑器编写, 毫无疑问, 不行的. 今天用python写了个脚本, 实现它.

<!-- more -->

### 主要思路

* 检测**BINLOG**到**BINLOG**段的行数放入一个集合中, 变量匹配, 如果集合中有删除`delete`, 字段, 则不写入到结果文件中.一下给出的是样列数据, 需要处理包含delete的BINLOG区域部分

    ```sql
        SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
        # at 21479
        #160925  0:06:49 server id 1  end_log_pos 21558 CRC32 0x086f6402 	Query	thread_id=8	exec_time=0	error_code=0
        SET TIMESTAMP=1474733209/*!*/;
        /*!\C utf8 *//*!*/;
        SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=192/*!*/;
        BEGIN
        /*!*/;
        # at 21558
        #160925  0:06:49 server id 1  end_log_pos 21628 CRC32 0x008a106e 	Table_map: `data_backup`.`goods_feature` mapped to number 114
        # at 21628
        #160925  0:06:49 server id 1  end_log_pos 21782 CRC32 0x1c9d6f66 	Delete_rows: table id 114 flags: STMT_END_F

        BINLOG '
        maTmVxMBAAAARgAAAHxUAAAAAHIAAAAAAAEAC2RhdGFfYmFja3VwAA1nb29kc19mZWF0dXJlAAQD
        Aw8PBGAAYAAAbhCKAA==
        maTmVyABAAAAmgAAABZVAAAAAHIAAAAAAAEAAgAE//ABAAAAAAAAAAbpnovlrZAA8AUAAAAAAAAA
        BuiinOWtkADwBwAAAAAAAAAG5aSW5aWXAPAMAAAAAAAAAAbnp4voo6QA8BAAAAAAAAAABuevrueQ
        gwDwFgAAAAAAAAAG6Laz55CDAPAaAAAAAAAAAAbmjpLnkIMAZm+dHA==
        '/*!*/;
        ### DELETE FROM `data_backup`.`goods_feature`
        ### WHERE
        ###   @1=1
        ###   @2=0
        ###   @3='鞋子'
        ###   @4=''
        ### DELETE FROM `data_backup`.`goods_feature`
        ### WHERE
        ###   @1=5
        ###   @2=0
        ###   @3='袜子'
        ###   @4=''
        ### DELETE FROM `data_backup`.`goods_feature`
        ### WHERE
        ###   @1=7
        ###   @2=0
        ###   @3='外套'
        ###   @4=''
        ### DELETE FROM `data_backup`.`goods_feature`
        ### WHERE
        ###   @1=12
        ###   @2=0
        ###   @3='秋裤'
        ###   @4=''
        ### DELETE FROM `data_backup`.`goods_feature`
        ### WHERE
        ###   @1=16
        ###   @2=0
        ###   @3='篮球'
        ###   @4=''
        ### DELETE FROM `data_backup`.`goods_feature`
        ### WHERE
        ###   @1=22
        ###   @2=0
        ###   @3='足球'
        ###   @4=''
        ### DELETE FROM `data_backup`.`goods_feature`
        ### WHERE
        ###   @1=26
        ###   @2=0
        ###   @3='排球'
        ###   @4=''
        # at 21782
        #160925  0:06:49 server id 1  end_log_pos 21813 CRC32 0x83e74fc6 	Xid = 451
        COMMIT/*!*/;
        # at 21813
        #160925  0:07:27 server id 1  end_log_pos 21878 CRC32 0x5e00f8b9 	Anonymous_GTID	last_committed=67	sequence_number=68
        SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
        # at 21878
        #160925  0:07:27 server id 1  end_log_pos 21957 CRC32 0x34843f3a 	Query	thread_id=8	exec_time=0	error_code=0
        SET TIMESTAMP=1474733247/*!*/;
        /*!\C utf8mb4 *//*!*/;
        SET @@session.character_set_client=45,@@session.collation_connection=45,@@session.collation_server=192/*!*/;
        BEGIN
        /*!*/;
        # at 21957
        #160925  0:07:27 server id 1  end_log_pos 22027 CRC32 0xa31bc0f3 	Table_map: `data_backup`.`goods_feature` mapped to number 114
        # at 22027
        #160925  0:07:27 server id 1  end_log_pos 22088 CRC32 0x9195b521 	Write_rows: table id 114 flags: STMT_END_F

        BINLOG '
        v6TmVxMBAAAARgAAAAtWAAAAAHIAAAAAAAEAC2RhdGFfYmFja3VwAA1nb29kc19mZWF0dXJlAAQD
        Aw8PBGAAYAAA88Abow==
        v6TmVx4BAAAAPQAAAEhWAAAAAHIAAAAAAAEAAgAE//AeAAAAAAAAAA/nrKzkuozlpKnllYblk4EA
        IbWVkQ==
        '/*!*/;
        ### INSERT INTO `data_backup`.`goods_feature`
        ### SET
        ###   @1=30
        ###   @2=0
        ###   @3='第二天商品'
        ###   @4=''
        # at 22088
        #160925  0:07:27 server id 1  end_log_pos 22119 CRC32 0x69383b9d 	Xid = 457
        COMMIT/*!*/;
        # at 22119
        #160925  0:07:27 server id 1  end_log_pos 22184 CRC32 0xb7f8d30d 	Anonymous_GTID	last_committed=68	sequence_number=69
        SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
        # at 22184
        #160925  0:07:27 server id 1  end_log_pos 22263 CRC32 0x104743ca 	Query	thread_id=8	exec_time=0	error_code=0
        SET TIMESTAMP=1474733247/*!*/;
        BEGIN
        /*!*/;
        # at 22263
        #160925  0:07:27 server id 1  end_log_pos 22333 CRC32 0x4cd5ba44 	Table_map: `data_backup`.`goods_feature` mapped to number 114
        # at 22333
        #160925  0:07:27 server id 1  end_log_pos 22396 CRC32 0xdb3caddc 	Write_rows: table id 114 flags: STMT_END_F

    ```
* 直接贴码了, 使用命令`python cutdelete.py --input=backup.sql --output=/path/to/result.sql` **python** 好久没用了, 代码写的一般, 欢迎拍砖.

    ```python
        #!/usr/bin/env python
        #-*- encoding: utf-8 -*-
        # author luowen<bigpao.luo@gmail.com>

        import argparse, re, time
        from functools import reduce


        class CutDelete():
            """
                将backup.sql中的delete语句删除处理脚本
            """
            PREFIX = "BINLOG '"
            listOfLine = []

            def __init__(self, inputFilename, outputFilename):
                self.inputFilename = inputFilename
                self.outputFilename = outputFilename

            def writeOutputFile(self):
                fileHandle = open(self.outputFilename, 'a+', encoding="utf-8")
                if len(self.listOfLine) > 0:
                    rePatternOfDelete = re.compile(".*DELETE.*", re.IGNORECASE)
                    isSkip = False
                    for line in self.listOfLine:
                        if rePatternOfDelete.match(line):
                            isSkip = True
                            break;
                    if not isSkip:
                        fileHandle.writelines(self.listOfLine)
                    self.listOfLine = [] # reset list of line
                fileHandle.close()

            def main(self):
                with open(self.inputFilename, "r", encoding="utf-8") as fileHandle:
                    for line in fileHandle:
                        if line.startswith(self.PREFIX):
                            self.writeOutputFile()
                        self.listOfLine.append(line)
                return True

        class InputArgsParser():
            """
                获取输入文件, 和写出文件
            """

            def __init__(self):
                self.parser = argparse.ArgumentParser(description="获取输入文件和输出文件")

            def getArgs(self):
                self.parser.add_argument("--input", type=str, required=True, help="需要处理的文件")
                self.parser.add_argument("--output", type=str, required=True, help="处理后的文件名")
                return self.parser.parse_args()



        if __name__ == "__main__":

            inputArgsParser = InputArgsParser()
            args =  inputArgsParser.getArgs()

            doCutDeleteObj = CutDelete(args.input, args.output)
            resultSet = doCutDeleteObj.main()
    ```

* 有兴趣可以使用php的`yield, yield -> send` 实现, 也是不错的选择!
