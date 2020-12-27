# Fluent Bit with CRI Log and JSON

With `dockerd` deprecated as a Kubernetes container runtime, we moved to `containerd`. After the change, our `fluentbit` logging didn't parse our JSON logs correctly. containerd uses the `CRI Log` format which is slightly different and requires additional parsing to parse JSON application logs.

We couldn't find a good end-to-end example, so we created this from various GitHub issues. There are some features missing (like multi-line logs) and we love PRs.

## Sample Config

[config.yaml](./config.yaml) contains a complete and minimal example configuration using `stdout`. We have tested with `stdout` and `Azure Log Analytics`. While not tested, it should work with `Elastic Search` and outher `output` providers as well.

> You will need to change the `output` `match` from `myapp*.*`

### Log Changes

> Note - there are several GitHub discussions on the challenges with multi-line CRI Logs

In [config](./config.yaml) there are three changes:

- Add the CRI parser which is a regex parser that maps the CRI Log fields into `time` `stream` `logtag` and `message`
  - `time` and `stream` map to existing `dockerd` log fields
  - `message` contains the text of the message, which, in our case is JSON

```yaml

[PARSER]
    Name        cri
    Format      regex
    Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<message>.*)$
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L%z

```

- Add a `filter` that parses the JSON from the `message` field

```yaml

[FILTER]
    Name      cri
    Match     kube.*
    Key_Name  message
    Parser    json

```

- Change the `Parser` on the input from `json` (or `docker`) to the `cri` parser

```yaml

[INPUT]
    Name              tail
    Tag               kube.*
    Path              /var/log/containers/*.log
    Parser            cri

```

### Attribution

We copied a lot of the yaml from several different GitHub issues. Thank you!

## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us the rights to use your contribution. For details, visit <https://cla.opensource.microsoft.com>

When you submit a pull request, a CLA bot will automatically determine whether you need to provide a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Trademarks

This project may contain trademarks or logos for projects, products, or services.

Authorized use of Microsoft trademarks or logos is subject to and must follow [Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).

Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.

Any use of third-party trademarks or logos are subject to those third-party's policies.
