����ͨ��HTTP״̬��һ��������ʹ����һ������״̬������������ͬ�������ｫ��״̬�������Э��״̬������ȥ�˷�����״̬����ʹ�������̵Ŀɶ��ԡ���ά���Զ������ߡ�������ջЭ�̵������൱�ã�����Ҳ���Ų�������ܡ�

����cpp20Э�̵�ʹ�ý̳��Ѿ��ܶ��ˣ����ﲻ��������﷨�ˡ���Ŀǰ��׼��ȱ��Э��������ȼ�дһ��`promise_type`��������Ҫʹ�õ���`co_return`��`co_yeild`������ʵ����`promise_type`��`return_value`��`yield_value`��

```cpp
template<class T>
struct Task {
    struct promise_type;
    using handle = std::coroutine_handle<promise_type>;
    struct promise_type {
        ...
        void return_value(T val) {
            m_val = val;
        }
        std::suspend_always yield_value(T val) {
            m_val = val;
            return {};
        }
        ...
        T m_val;
    };
    ...
    T get() {
        return m_handle.promise().m_val;
    }
    void resume() {
        m_handle.resume();
    }
    ...
private:
    handle m_handle = nullptr;
};
```


��Ȼ��״̬��ʹ��Э����������ʽ��״̬������״̬��������Ҫ״̬ת�ƣ����������µ���״̬
```cpp
/// ��״̬����״̬
enum CheckState{
	NO_CHECK,		// ��ʼ״̬
	CHECK_LINE,		// ���ڽ���������
	CHECK_HEADER,	// ���ڽ�������ͷ
	CHECK_CHUNK		// ����chunk
};
```
���������
```cpp
 /// ������
enum Error {
	NO_ERROR = 0,		// ��������
	INVALID_METHOD,		// ��Ч�� Method
	INVALID_PATH,		// ��Ч�� Path 
	INVALID_VERSION,	// ��Ч�� Version
	INVALID_LINE,		// ��Ч��������
	INVALID_HEADER,		// ��Ч������ͷ
	INVALID_CODE,		// ��Ч�� Code
	INVALID_REASON,		// ��Ч�� Reason
	INVALID_CHUNK		// ��Ч�� Chunk
};
```

����Request��Response�ĸ�ʽһ�����������У�����ͷ�����ģ�������������״̬��һ������parse_line��parse_header�����麯����̬��ʵ�ֲ�ͬ�Ĵ�״̬����

```cpp
// ��״̬��
size_t HttpParser::execute(char *data, size_t len, bool chunk) {
    ...
    size_t offset = 0;
    size_t i = 0;
    switch (m_checkState) {
        case NO_CHECK: {
            m_checkState = CHECK_LINE;
            m_parser = parse_line();
        }
            // ����������
        case CHECK_LINE: {
			// �������е�ÿ���ַ�����
            for(; i < len; ++i){
                m_cur = data + i;
                m_parser.resume();
                m_error = m_parser.get();
                ++offset;
                if(isFinished()){
                    memmove(data, data + offset, (len - offset));
                    return offset;
                }
                if(m_checkState == CHECK_HEADER){
                    ++i;
                    m_parser = parse_header();
                    break;
                }
            }
            if(m_checkState != CHECK_HEADER) {
                memmove(data, data + offset, (len - offset));
                return offset;
            }
        }
        // ��������ͷ
        case CHECK_HEADER: {
			// ������ͷ��ÿ���ַ�����
            for(; i < len; ++i){
                m_cur = data + i;
                m_parser.resume();
                m_error = m_parser.get();
                ++offset;
                if(isFinished()){
                    memmove(data, data + offset, (len - offset));
                    return offset;
                }
            }
            break;
        }
        ...
    }
    memmove(data, data + offset, (len - offset));
    return offset;
}
```

���ǽ����Ļص��������������Ĵ�״̬������

```cpp
// Request�ص�
void on_request_method(const std::string& str);
void on_request_path(const std::string& str);
void on_request_query(const std::string& str);
void on_request_fragment(const std::string& str) ;
void on_request_version(const std::string& str);
void on_request_header(const std::string& key, const std::string& val);
void on_request_header_done();
// Response�ص�
void on_response_version(const std::string& str);
void on_response_status(const std::string& str);
void on_response_reason(const std::string& str);
void on_response_header(const std::string& key, const std::string& val);
void on_response_header_done();
void on_response_chunk(const std::string& str);
```

����������Э��״̬������Ҫ���֣�����ǽ���Request�������еĴ�״̬�������Կ������������̷ǳ�������ÿ������һ���־ͻᴥ����Ӧ�Ļص�����ȡ�������ݡ��ص������parse_line()ÿ��resume��������һ�ν������̵���һ������ΪЭ�̱������һ��״̬��������Э�̾���������ʽ״̬ת�ƣ�ʹ�ý����������ᣬ�߼�������

```cpp
/**
* @brief ���� HTTP �����������
*/
Task<HttpParser::Error> HttpRequestParser::parse_line() {
    std::string buff;
    // ��ȡmethod
    while(isalpha(*m_cur)) {
        buff.push_back(*m_cur);
        co_yield NO_ERROR;
    }
    if(buff.empty()) {
        co_return INVALID_METHOD;
    }
    if(*m_cur != ' ') {
        // method֮���ǿո񣬸�ʽ����
        co_return INVALID_METHOD;
    } else {
        // ����method, �����ص�����
        on_request_method(buff);
        if (m_error) {
            co_return INVALID_METHOD;
        }
        buff = "";
        co_yield NO_ERROR;
    }
    // ��ȡ·��
    while(std::isprint(*m_cur) && strchr(" ?", *m_cur) == nullptr) {
        buff.push_back(*m_cur);
        co_yield NO_ERROR;
    }
    if(buff.empty()) {
        // pathΪ�գ���ʽ����
        co_return INVALID_PATH;
    }

    if(*m_cur == '?'){
        on_request_path(buff);
        buff = "";
        co_yield NO_ERROR;
        // ��ȡquery
        while(std::isprint(*m_cur) && strchr(" #", *m_cur) == nullptr) {
            buff.push_back(*m_cur);
            co_yield NO_ERROR;
        }

        if(*m_cur == '#'){
            on_request_query(buff);
            buff = "";
            // ��ȡfragment
            while(std::isprint(*m_cur) && strchr(" ", *m_cur) == nullptr) {
                buff.push_back(*m_cur);
                co_yield NO_ERROR;
            }
            if(*m_cur != ' '){
                // fragment֮���ǿո񣬸�ʽ����
                co_return INVALID_PATH;
            } else {
                on_request_fragment(buff);
                buff = "";
                co_yield NO_ERROR;
            }

        } else if(*m_cur != ' '){
            // query֮���ǿո񣬸�ʽ����
            co_return INVALID_PATH;
        } else {
            on_request_query(buff);
            buff = "";
            co_yield NO_ERROR;
        }
    } else if(*m_cur != ' '){
        // path֮���ǿո񣬸�ʽ����
        co_return INVALID_PATH;
    } else {
        on_request_path(buff);
        buff = "";
        co_yield NO_ERROR;
    }
    const char* version = "HTTP/1.";

    while (*version) {
        if(*m_cur != *version) {
            co_return INVALID_VERSION;
        }
        version++;
        co_yield NO_ERROR;
    }
    if(*m_cur != '1' && *m_cur != '0') {
        co_return INVALID_VERSION;
    } else {
        buff="1.";
        buff.push_back(*m_cur);
        on_request_version(buff);
        if (m_error) {
            co_return INVALID_VERSION;
        }
        buff = "";
        co_yield NO_ERROR;
    }
    if(*m_cur != '\r') {
        co_return INVALID_LINE;
    }
    co_yield NO_ERROR;
    if(*m_cur != '\n') {
        co_return INVALID_LINE;
    }
    // ״̬ת��
    m_checkState = CHECK_HEADER;
    co_return NO_ERROR;
}
```

����Request������ͷ

```cpp
/**
* @brief ���� HTTP ���������ͷ
*/
Task<HttpParser::Error> HttpRequestParser::parse_header() {

    std::string key, val;
    // ѭ����ȡheader��ֱ����ȡ��\r\n\r\nʱ���
    while (!isFinished()){
        // ��ȡkey
        while(std::isprint(*m_cur) && *m_cur != ':') { // ��ȡ�����������ַ��洢��key�С�
            key.push_back(*m_cur);
            co_yield NO_ERROR;
        }
        // ��ȡ:
        if(*m_cur != ':') {
            co_return INVALID_HEADER;
        } else {
            co_yield NO_ERROR;
        }
        // ��ȡ�ո�
        while(*m_cur == ' ') {
            co_yield NO_ERROR;
        }

        // ��ȡvalue
        while (std::isprint(*m_cur)) {
            val.push_back(*m_cur);
            co_yield NO_ERROR;
        }
        if(*m_cur != '\r') {
            co_return INVALID_HEADER;
        }
        co_yield NO_ERROR;
        if(*m_cur != '\n') {
            co_return INVALID_HEADER;
        }
        co_yield NO_ERROR;
        if(*m_cur == '\r') {
            co_yield NO_ERROR;
            // �ж��ǲ��� \r\n\r\n
            if(*m_cur == '\n'){
                on_request_header(key, val);
                on_request_header_done();
                m_finish = true;
                // ״̬ת��
                m_checkState = NO_CHECK;
                //yield;
                //���˳�ѭ��
            } else {
                co_return INVALID_HEADER;
            }
        } else {
            on_request_header(key, val);
            key.clear();
            val.clear();
            key.push_back(*m_cur);
            co_yield NO_ERROR;
        }
    }
    // ״̬ת��
    m_checkState = NO_CHECK;
    co_return NO_ERROR;
}
```

����Response��������

```cpp
/**
* @brief ���� HTTP ��Ӧ��������
*/
Task<HttpParser::Error> HttpResponseParser::parse_line() {
    std::string buff;
    const char* version = "HTTP/1.";
    // �ж�HTTP�汾�Ƿ�Ϸ�
    while (*version) {
        if(*m_cur != *version) {
            co_return INVALID_VERSION;
        }
        version++;
        co_yield NO_ERROR;
    }

    if(*m_cur != '1' && *m_cur != '0') {
        co_return INVALID_VERSION;
    } else {
        buff="1.";
        buff.push_back(*m_cur);
        on_response_version(buff);
        if (m_error) {
            co_return INVALID_VERSION;
        }
        buff = "";
        co_yield NO_ERROR;
    }
    // �����ո�
    while(*m_cur == ' ') {
        co_yield NO_ERROR;
    }
    // ��ȡstatus
    while(isdigit(*m_cur)) { // ��ȡ�����������ַ��洢��method_�С�
        buff.push_back(*m_cur);
        co_yield NO_ERROR;
    }
    // ������method
    if(buff.empty()) {
        co_return INVALID_CODE;
    }
    // ��ȡ�ո�
    if(*m_cur != ' ') {
        // status֮���ǿո񣬸�ʽ����
        co_return INVALID_CODE;
    } else {
        // ����status, �����ص�����
        on_response_status(buff);
        buff = "";
        co_yield NO_ERROR;
    }
    // �����ո�
    while(*m_cur == ' ') {
        co_yield NO_ERROR;
    }
    // ��ȡReason
    while(std::isalpha(*m_cur) || *m_cur == ' ') {
        buff.push_back(*m_cur);
        co_yield NO_ERROR;
    }
    if(buff.empty()) {
        // pathΪ�գ���ʽ����
        co_return INVALID_REASON;
    }

    if(*m_cur != '\r') {
        co_return INVALID_LINE;
    }
    co_yield NO_ERROR;
    if(*m_cur != '\n') {
        co_return INVALID_LINE;
    }
    on_response_reason(buff);
    // ״̬ת��
    m_checkState = CHECK_HEADER;
    co_return NO_ERROR;
}
```

����Response������ͷ

```cpp
/**
* @brief ���� HTTP ��Ӧ������ͷ
*/
Task<HttpParser::Error> HttpResponseParser::parse_header() {
    std::string key, val;
    // ѭ����ȡheader��ֱ����ȡ��\r\n\r\nʱ���
    while (!isFinished()){
        // ��ȡkey
        while(std::isprint(*m_cur) && strchr(":", *m_cur) == nullptr) {
            key.push_back(*m_cur);
            co_yield NO_ERROR;
        }
        // ��ȡ:
        if(*m_cur != ':') {
            co_return INVALID_HEADER;
        } else {
            co_yield NO_ERROR;
        }
        // ��ȡ�ո�
        while(*m_cur == ' ') {
            co_yield NO_ERROR;
        }

        // ��ȡvalue
        // ��ȡ�����������ַ��洢��val�С�
        while (std::isprint(*m_cur)) {
            val.push_back(*m_cur);
            co_yield NO_ERROR;
        }
        if(*m_cur != '\r') {
            co_return INVALID_HEADER;
        }
        co_yield NO_ERROR;
        if(*m_cur != '\n') {
            co_return INVALID_HEADER;
        }
        co_yield NO_ERROR;
        if(*m_cur == '\r') {
            co_yield NO_ERROR;
            // �ж��ǲ��� \r\n\r\n
            if(*m_cur == '\n'){
                on_response_header(key, val);
                on_response_header_done();
                m_finish = true;
                // ״̬ת��
                m_checkState = NO_CHECK;
                //co_yield NO_ERROR;
                //���˳�ѭ��
            } else {
                co_return INVALID_HEADER;
            }
        } else {
            on_response_header(key, val);
            key.clear();
            val.clear();
            key.push_back(*m_cur);
            co_yield NO_ERROR;
        }
    }
    // ״̬ת��
    m_checkState = NO_CHECK;
    co_return NO_ERROR;
}
```

���ϵĽ�����������ͬС�죬ֻҪ�����һ��Э��״̬������ͻᷢ���κε�״̬�����������Ը�д��Э����ʽ�ġ�

����ͷ����Ϣ�󣬾Ϳ���ͨ��ContentLength����ȡ���ĵ������ˣ�������Э��������¡�

```cpp
HttpRequest::ptr HttpSession::recvRequest() {
    HttpRequestParser::ptr parser(new HttpRequestParser);
    uint64_t buff_size = HttpRequestParser::GetHttpRequestBufferSize();
    std::string buffer;
    buffer.resize(buff_size);
    char* data = &buffer[0];
    size_t offset = 0;
    while (!parser->isFinished()) {
        ssize_t len = read(data + offset, buff_size - offset);
        if (len <= 0) {
            close();
            return nullptr;
        }
        len += offset;
        size_t nparse = parser->execute(data, len);
        if (parser->hasError() || nparse == 0) {
            ACID_LOG_DEBUG(g_logger) << "parser error code:" << parser->hasError();
            close();
            return nullptr;
        }
        offset = len - nparse;
    }
    uint64_t length = parser->getContentLength();
    if (length >= 0) {
        std::string body;
        body.resize(length);
        size_t len = 0;
        if (length >= offset) {
            memcpy(&body[0], data, offset);
            len = offset;
        } else {
            memcpy(&body[0], data, length);
            len = length;
        }
        length -= len;
        if(length > 0) {
            if(readFixSize(&body[len], length) <= 0) {
                close();
                return nullptr;
            }
        }
        parser->getData()->setBody(body);
    }
    return parser->getData();
}
```

## ���

͸�����󿴱��ʣ�Э�̵ײ�ͨ��Э��֡�������������ģ�״̬��Ҳ�����������棬ʹ�����ǲ�����ʽ����״̬����ȥ�˷�����״̬ת������ͬ������˵��Э��֮��״̬������ݹ�֮��ջ��

Э�̵�ʹ�ó��ϲ�ֹ�첽������һ��ǿ���������ֵ�����Ǹ�����ȥ�˽⡣

����Ȥ���������������Ĵ��� https://github.com/zavier-wong/acid/blob/main/source/http/parse.cpp