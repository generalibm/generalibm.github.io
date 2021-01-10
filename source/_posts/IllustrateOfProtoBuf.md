---
title: Protobuf 101
date: 2016-03-16 03:33:18
tags: protobuf
categories: Tech
toc: true
---

## Protobuf（google protocol buffer）

#### 一、简介

Google Protocol Buffer( 简称 Protobuf) 是 Google 公司内部的混合语言数据标准，目前已经正在使用的有超过 48,162 种报文格式定义和超过 12,183 个 .proto 文件。他们用于 RPC 系统和持续数据存储系统。
Protocol Buffers 是一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或 RPC 数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。官方提供了` C++`、`Java`、`Python`、`go`、`c #` 五种语言的 API，因为开源所以其他语言也相继有了支持。[详情可查看](https://github.com/google/protobuf/releases)。

<!--more-->

#### 二、安装

环境：Ubuntu 15.04 
直接下载安装包，解压后编译安装：
下载地址：[站点1](https://github.com/google/protobuf/releases)，或者[站点2](https://developers.google.com/protocol-buffers/docs/downloads)
解压：`tar –xzvf protobuf-3.0.0-beta-3.tar.gz`
安装依赖工具：`sudo apt-get install autoconf automake libtool curl make g++ unzip`
编译安装[4]：
`./autogen.sh`
`./configure`
`make check`
`sudo make install`
`sudo ldconfig # refresh shared library cache.`

#### 三、例子程序

##### 3.1 Google官方例子[1]

1 . 编写文件`addressbook.proto`

```protobuf
//syntax = "proto2";

package tutorial;

message Person {
	required string name = 1;
	required int32 id = 2;
	optional string email = 3;

	enum PhoneType {
		MOBILE = 0;
		HOME = 1;
		WORK = 2;
	}

	message PhoneNumber {
		required string number = 1;
		optional PhoneType phone_type = 2 [default = HOME];
	}

	repeated PhoneNumber phone = 4;
}

message AddressBook {
	repeated Person person = 1;	
}

```

2 . 编译生成头文件和实现文件
`protoc –I=./ --cpp_out=./ ./addressbook.proto`

3 . 文件`writer.cc`

```c++
#include <iostream>
#include <fstream>
#include <string>

#include "addressbook.pb.h"

using namespace std;

void PromptForAddress(tutorial::Person * person) {
	cout << "Enter person ID name :";
	int id;
	cin >> id;
	person->set_id(id);
	cin.ignore(256, '\n');

	cout << "Enter name:";
	getline(cin , *person->mutable_name());

	cout << "Enter email address (blank for none):";
	string email;
	getline(cin, email);
	if (!email.empty()) {
		person->set_email(email);
	}

	while (true) {
		cout << "Enter a phone number (or leave a blank to finish):";
		string number;
		getline(cin, number);
		if (number.empty()) {
			break;
		}

		tutorial::Person::PhoneNumber * phone_number = person->add_phone();
		phone_number->set_number(number);

		cout << "Is this a mobile, home or work phone?";
		string type;
		getline(cin, type);
		if ("mobile" == type) {
			phone_number->set_phone_type(tutorial::Person::MOBILE);
		} else if ("home" == type) {
			phone_number->set_phone_type(tutorial::Person::HOME);
		} else if ("work" == type) {
			phone_number->set_phone_type(tutorial::Person::WORK);
		} else {
			cout << "Unknow phone type." << endl;
		}// end else

	}// end while
}

int main(int argc, char *argv[])
{
	//
	GOOGLE_PROTOBUF_VERIFY_VERSION;

	if (argc != 2) {
		cerr << "Usage :" << argv[0] << "ADDRESS_BOOK_FILE" << endl;
		return -1;
	}
	
	tutorial::AddressBook address_book;
	{
		fstream input(argv[1], ios::in | ios::binary);
		if (!input) {
			cout << argv[1] << ":File not found. Creating a new file." << endl;
		} else if (!address_book.ParseFromIstream(&input)) {
			cerr << "Failed to parse address book." << endl;
			return -1;
		}
	}

	// Add an address.
	PromptForAddress(address_book.add_person()); 

	{
		// Write the new address book back to disk.
		fstream output(argv[1], ios::out | ios::trunc | ios::binary);
		if (!address_book.SerializeToOstream(&output)) {
			cerr << "Failed to write address book." << endl;
			return -1;
		}//end if
	}

	// Delete all global objects allocated by libprotobuf.
	google::protobuf::ShutdownProtobufLibrary();

	return 0;
}

```

4 . 文件`reader.cc`

```c++
#include <iostream>
#include <fstream>
#include <string>

#include "addressbook.pb.h"

using namespace std;

void ListPeople(const tutorial::AddressBook & address_book) {
	for (int i = 0; i < address_book.person_size(); i++) {
		const tutorial::Person & person = address_book.person(i);

		cout << " Person ID: " << person.id() << endl;
		cout << " Name : " << person.name() << endl;
		if (person.has_email()) {
			cout << " Email : " << person.email() << endl;
		}
		
		for (int j = 0; j < person.phone_size(); j++) {
			const tutorial::Person::PhoneNumber & phone_number = person.phone(j);
			
			switch (phone_number.phone_type()) {
				case tutorial::Person::MOBILE:
					cout << " MOBILE : ";
					break;
				case tutorial::Person::HOME:
					cout << " HOME : ";
					break;
				case tutorial::Person::WORK:
					cout << " WORK : ";
					break;
			}
			
			cout << phone_number.number() << endl;
		}

	}// end for
}



int main(int argc, char *argv[])
{
	//::GOOGLE::PROTOBUF_VERIFY_VERSION;

	if (argc != 2) {
		cerr << "Usage: " << argv[0] << " ADDRESS_BOOK_FILE" << endl;
		return -1;
	}	
	tutorial::AddressBook address_book;

	{
		fstream input(argv[1], ios::in | ios::binary);
		if (!address_book.ParseFromIstream(&input)) {
			cerr << "Failed to parse address book." << endl;
			return -1;
		}
	}
	
	ListPeople(address_book);

	google::protobuf::ShutdownProtobufLibrary();

	return 0;
}

```

5 . 编译
`g++ -o writer writer.cc addressbook.pb.cc –lprotobuf`
`g++ -o reader reader.cc addressbook.pb.cc –lprotobuf`
6 . 执行结果
`./writer mid_file`
`$Enter person ID name :1001`
`$Enter name :Yun Ma`
`$Enter email address (blank for none):ali@ali.com`
`$Enter a phone number (or leave a blank to finish):17088888888`
`$Is this a mobile , home or work phone?work`

`./reader mid_file`
```bash
Person ID :1001
Name : Yun MA
Email : ali@ali.com
WORK : work
```

##### 3.2 DIY简单helloworld例子[3]

1．编写`lm.helloworld.proto`文件

```protobuf
syntax = "proto2";

package lm;

message helloworld {
	required int32		id = 1;
	required string		str = 2;
	optional int32		opt = 3;
}

```
2．编译生成头文件及相应实现文件
```bash
protoc -I=./ --cpp_out=./ ./lm.helloword.proto
```
3．编写writer.cc
```c++
#include <iostream>
#include <fstream>

#include "lm.helloworld.pb.h"

using std::cout;
using std::cerr;
using std::endl;
using std::fstream;
using std::ios;

int main()
{
	lm::helloworld msg1;
	msg1.set_id(1001);
	msg1.set_str("google");

	fstream output("./log", ios::out | ios::trunc | ios::binary);

	if (!msg1.SerializeToOstream(&output)) {
		cerr << "Failed to write msg." << endl;
		return -1;
	}
	
	return 0;
}

```
4．编写reader.cc
```c++
#include "lm.helloworld.pb.h"

#include <iostream>
#include <fstream>

using std::cout;
using std::endl;
using std::fstream;
using std::ios;
using std::cerr;

void listMsg(const lm::helloworld & msg) {
	cout << "id = " << msg.id() << endl;
	cout << "str = " << msg.str() << endl;
}


int main()
{
	lm::helloworld msg1;
#if 1
	{
		fstream input("./log", ios::in | ios::binary);
		if (!msg1.ParseFromIstream(&input)) {
			cerr << "Failed to parse address book." << endl;
			return -1;
		}
	}
#endif	
	listMsg(msg1);

	return 0;
}

```
5 .  编译执行
`./writer`
`$cat log`
`$理google`

`./reader`
`$ id = 1001`
`$ str = google`

#### 四、参考资料

[1]https://developers.google.com/protocol-buffers/docs/cpptutorial#defining-your-protocol-format
[2]https://github.com/google/protobuf/blob/master/src/README.md
[3]http://www.ibm.com/developerworks/cn/linux/l-cn-gpb/


