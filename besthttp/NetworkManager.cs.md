```cs
private static HTTPRequest GetWebRequest(HTTPMethods _methodType, string _uri, Dictionary<string, string> _data = null)
{
    switch (_methodType)
    {
        case HTTPMethods.Get:
            return HTTPRequest.CreateGet(_uri);
        case HTTPMethods.Post:
            var request = HTTPRequest.CreatePost(_uri);
            var stream = new MultipartFormDataStream();
            foreach (var kv in _data)
            {
                stream.AddField(kv.Key, kv.Value);
            }

            request.UploadSettings.UploadStream = stream;

            return request;
        default:
            return null;
    }
}
```

```cs
private IEnumerator CoroutineRequestProcess<T>(string _uri, object _data, Action<T> _success, Action<ErrorCode> _fail, params Type[] _addType) where T : IResponseCommon
{
    // 큐에 대기
    yield return new WaitForEndOfFrame();

    // https://bestdocshub.pages.dev/HTTP/getting-started/#callbacks
    T result = null;

    var method = null == _data
        ? HTTPMethods.Get
        : HTTPMethods.Post;

    var dic = GetDataToDic(_data);

    var url = _uri;
    if (_data is IRequestCommon common)
    {
        url = $"{url}?{nameof(IRequestCommon.token)}={common.token}&{nameof(IRequestCommon.sq)}={GenSeqNumber}";
        dic.Remove(nameof(IRequestCommon.token));
        dic.Remove(nameof(IRequestCommon.sq));
    }

    for (var i = 0; i < RETRY_MAXCOUNT; i++)
    {
        var q = GetWebRequest(method, url, dic);

        lastRequestProtocolURL = q.Uri.ToString();
        lastRequestProtocolTime = Time.time;

        q.Send();

        yield return q;

        lastRequestProtocolTime = Time.time - lastRequestProtocolTime;
        lastRequestProtocolException = q.Exception;

        if (q.State == HTTPRequestStates.Finished)
        {
            if (q.Response is { IsSuccess: true })
            {
                var text = q.Response.DataAsText;
                if (null != _addType)
                {
                    if (_addType.Length > 0)
                    {
                        foreach (var t in _addType)
                        {
                            if (Deserialize(t))
                            {
                                break;
                            }
                        }
                    }
                    else
                    {
                        result = JsonConvert.DeserializeObject<T>(text, _jsonConvertSettings);
                    }
                }
                else
                {
                    result = JsonConvert.DeserializeObject<T>(text, _jsonConvertSettings);
                }

                if (result == null)
                {
                    Player_ErrorReport($"DataAsText={text}");
                    continue;
                }

                CheckBalanceUpdated(result);

                if (IsSuccess(result.result))
                {
                    _success?.Invoke(result);
                    yield break;
                }

                bool Deserialize(Type t)
                {
                    if (null != q.Response.GetFirstHeaderValue(t.Name))
                    {
                        result = JsonConvert.DeserializeObject(q.Response.DataAsText, t) as T;
                        return true;
                    }

                    return false;
                }
            }
            else
            {
                Player_ErrorReport($"StatusCode={q.Response.StatusCode} Message={q.Response.Message}");
            }
        }
        else
        {
            Player_ErrorReport($"State={q.State}");
        }

        yield return new WaitForSeconds(1);
    }

    if (null != result)
    {
        var code = (ErrorCode)result.result;
        ProtocolError(code);
        _fail?.Invoke(code);
    }
    else
    {
        ProtocolError(ErrorCode.HTTPERROR);
        _fail?.Invoke(ErrorCode.HTTPERROR);
    }

    yield return null;
}
```