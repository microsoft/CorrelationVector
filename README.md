
# Correlation Vector

## Background

**Correlation Vector** (a.k.a. **cV**) is a format and protocol standard for tracing and correlation of events through a distributed system based on a light weight vector clock.
The standard is widely used internally at Microsoft for first party applications and services and supported across multiple logging libraries and platforms (Services, Clients - Native, Managed, JS, iOS, Android etc). The standard powers a variety of different data processing needs ranging from distributed tracing & debugging to system and business intelligence, in various business organizations.

## Goals

- Make the standard specification externally available for implementation by non-Microsoft components or Open Source Microsoft components
- Support wire interop scenarios between Microsoft-internal and external tracing systems
- Describe the benefits of vector based tracing formats and to provide an exemplar vector based format for future standardization efforts with similar protocols

## Resources

- Design Constraints and Scenarios for the Correlation Vector: [Link](Scenarios.md)
- Protocol Specification:
  - [v2.1](cV%20-%202.1.md)
  - [v3.0](cV%20-%203.0.md) [In Review]
    - Support interop with W3C Distributed Tracing [Link](https://github.com/w3c/distributed-tracing)
    - Support for vector reset semantics to support traces of arbitrary depth
    - Support for versioning

## Reference Implementations

- [Cpp](https://github.com/Microsoft/CorrelationVector-Cpp) \(C\+\+\)
- [CSharp](https://github.com/Microsoft/CorrelationVector-CSharp) \(C\#\)
- [Go](https://github.com/Microsoft/CorrelationVector-Go)
- [Java](https://github.com/Microsoft/CorrelationVector-Java)
- [JavaScript](https://github.com/Microsoft/CorrelationVector-JavaScript)

## Future Roadmap

TBD

# Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

# Legal Notices

Microsoft and any contributors grant you a license to the Microsoft documentation and other content
in this repository under the [Creative Commons Attribution 4.0 International Public License](https://creativecommons.org/licenses/by/4.0/legalcode),
see the [LICENSE](LICENSE) file, and grant you a license to any code in the repository under the [MIT License](https://opensource.org/licenses/MIT), see the
[LICENSE-CODE](LICENSE-CODE) file.

Microsoft, Windows, Microsoft Azure and/or other Microsoft products and services referenced in the documentation
may be either trademarks or registered trademarks of Microsoft in the United States and/or other countries.
The licenses for this project do not grant you rights to use any Microsoft names, logos, or trademarks.
Microsoft's general trademark guidelines can be found at http://go.microsoft.com/fwlink/?LinkID=254653.

Privacy information can be found at https://privacy.microsoft.com/en-us/

Microsoft and any contributors reserve all others rights, whether under their respective copyrights, patents,
or trademarks, whether by implication, estoppel or otherwise.
