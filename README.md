# Fluent Bit with containerd, CRI-O and JSON

With `dockerd` deprecated as a Kubernetes container runtime, we moved to `containerd`. After the change, our `fluentbit` logging didn't parse our JSON logs correctly. `containerd` and `CRI-O` use the `CRI Log` format which is slightly different and requires additional parsing to parse JSON application logs.

We couldn't find a good end-to-end example, so we created this from various GitHub issues. There are some features missing (like multi-line logs) and we love PRs.

## Enhancement

The original version of this repo used a separate filter to parse the JSON. By changing the cri parser to use the `log` field instead of the `message` field, the `kubernetes filter` converts the JSON if `Merge_Log` is set to `On`

We also had to add to `lift` filters to the config to get the Kubernetes values to the root level.

## Sample Config

[config.yaml](./config.yaml) contains a complete and minimal example configuration using `stdout`. We have tested with `stdout` and `Azure Log Analytics`. While not tested, it should work with `Elastic Search` and outher `output` providers as well.

> You will need to change the `output` `match` from `myapp*.*`

### Config Changes

> Note - there are several GitHub discussions on the challenges with multi-line CRI Logs - additional processing is necessary and not included here

In [config](./config.yaml) there are three changes:

- Add the CRI parser which is a regex parser that maps the CRI Log fields into `time` `stream` `logtag` and `log`
  - `time` and `stream` map to existing `dockerd` log fields
  - `log` contains the text of the message, which, in our case is JSON
    - The JSON is parsed and merged in the `kubernetes filter`
      - `Merge_Log` must be set to `On`

> Note the regex expression uses `log` for the 4th field, not `message`

```yaml

[PARSER]
    Name        cri
    Format      regex
    Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L%z

```

- Add the `nest filters` to `lift` the Kubernetes values to the root document

```yaml

    [FILTER]
        Name          nest
        Match         kube.*
        Operation     lift
        Nested_under  kubernetes
        Add_prefix    kubernetes_

    [FILTER]
        Name          nest
        Match         kube.*
        Operation     lift
        Nested_under  kubernetes_labels
        Add_prefix    kubernetes_labels_

```

- Change the `Parser` on the input from `json` or `docker` to `cri`

```yaml

[INPUT]
    Name              tail
    Tag               kube.*
    Path              /var/log/containers/*.log
    Parser            cri

```

### Attribution

> Thank you!

We copied a lot of the yaml from several different GitHub issues / docs including [fluentbit docs](https://docs.fluentbit.io/manual/installation/kubernetes)

## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us the rights to use your contribution. For details, visit <https://cla.opensource.microsoft.com>

When you submit a pull request, a CLA bot will automatically determine whether you need to provide a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Trademarks

This project may contain trademarks or logos for projects, products, or services.

Authorized use of Microsoft trademarks or logos is subject to and must follow [Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).

Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.

Any use of third-party trademarks or logos are subject to those third-party's policies.
