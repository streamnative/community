# Unify “AuthAction” and “ResrouceOperations” in Pulsar Authorization Permissions

- Status: Proposal
- Feature Name: unify_pulsar_authorization_permission
- Propose Date: 2021-01-14
- RFC PR: [streamnative/community#0000](https://github.com/streamnative/community/pull/0000)
- Project Issue: [apache/pulsar#0000](https://github.com/apache/pulsar/issues/0000)
- Authors: Yong Zhang

# Motivation
Currently, Pulsar has two sets of authorization permissions. One is the AuthAction[1], which contains consume, produce, functions, sources, and sinks permission. You can grant those permissions to a "role" to restrict someone's operations on a topic or a namespace. And the other is "ResourceOperations", including TopicOperation[2], NamespaceOperation[3], and TenantOperation[4]. It provides a more granular permission control and basically covers all the operations in Pulsar.

But we have the following issues for current authorization service[5].

1. When you are granting a permission to a "role", it's using the "AuthAction" permissions by default. But when Pulsar service to check a role if has the required permissions, it will use the "ResrouceOperations" to check. That made pulsar service needs to match the "AuthActions" and the "ResourceOperations" for check a role has the required permission. And we don't provide a way to grant  "ResourceOperations" to a role. So I proposed we make all the operations on the "ResourceOperations" and deprecate the "AuthAction".

2. Another issue is we haven't managed topic level permissions with the authorization service. In the current implementation, the topic level permissions policies write to the zookeeper directly, we should manage them in the authorization service so it can be rewrite by different authorization plugins.

# Changes
## Authorization Permissions
First of all, we need to add the grant operations in the authorization provide to allow the authorization service has ability to grant the "ResourceOpeartions" to a role. So I will introduce the following methods in the current authorization provider:

```
public interface AuthorizationProvider extends Closeable {
    /**
     * Grant topic operation permissions on a topic to a role
     *
     * @param topic
     * @param role
     * @param topicOperation
     * @param authDataJson
     * @return
     */
    default CompletableFuture<Void> grantTopicOperationAsync(TopicName topic, String role, TopicOperation topicOperation, String authDataJson) {
        return FutureUtil.failedFuture(new UnsupportedOperationException("Unsupported operation"));
    }

    /**
     * Grant namespace operation permissions on a namespace to a role
     *
     * @param namespaceName
     * @param role
     * @param namespaceOperation
     * @param authDataJson
     * @return
     */
    default CompletableFuture<Void> grantNamespaceOperationAsync(NamespaceName namespaceName, String role, NamespaceOperation namespaceOperation, String authDataJson) {
        return FutureUtil.failedFuture(new UnsupportedOperationException("Unsupported operation"));
    }

    /**
     * Grant tenant operation permissions on a tenant to a role
     *
     * @param tenant
     * @param role
     * @param tenantOperation
     * @param authDataJson
     * @return
     */
    default CompletableFuture<Void> grantTenantOperationAsync(String tenant, String role, TenantOperation tenantOperation, String authDataJson) {
        return FutureUtil.failedFuture(new UnsupportedOperationException("Unsupported operation"));
    }
}
```

And then we need to change all the “AuthAction” to the “Resource Operation”, but for compatibility with the old version, we need to keep the "AuthAction" in the "ResourceOpeartion".
```
public enum TopicOperation {
    ...
    produce,
    consume,
    functions,
    sinks,
    sources
}
public enum NamespaceOperation {
    ...
    produce,
    consume,
    functions,
    sinks,
    sources
}

```

Validation operation already implemented, there is no changes for the validation methods.

## Toplc level Authorization Management
The topic level authorization all done in the PersistentTopicsBase[6] class and it doesn't use the authorization service to do the grant operation. We need to remove all the grant and revoke operation in the PersistentTopicBase and change it to use the authorization service. The permissions validation is using the authorization service to validate so there is no change for that.

# Compatibility
We need to ensure the “AuthAction” behaviours compatibility. As I mentioned in the changes, we will move the "AuthAction" into the "ResourceOpeartions" to ensure the compatibility with old versions.

# Test Plan
We need to add a group of tests for the new authorization permissions.
At the same time, the proposal is a improvement not a breaking change, we need to make sure the existent authorization tests can pass.

# Reference
[1] https://github.com/apache/pulsar/blob/4c6026213b743a7f23ae2a5a6d37ee7404b066db/pulsar-common/src/main/java/org/apache/pulsar/common/policies/data/AuthAction.java#L24
[2] https://github.com/apache/pulsar/blob/4c6026213b743a7f23ae2a5a6d37ee7404b066db/pulsar-common/src/main/java/org/apache/pulsar/common/policies/data/TopicOperation.java#L25
[3] https://github.com/apache/pulsar/blob/4c6026213b743a7f23ae2a5a6d37ee7404b066db/pulsar-common/src/main/java/org/apache/pulsar/common/policies/data/NamespaceOperation.java#L25
[4] https://github.com/apache/pulsar/blob/4c6026213b743a7f23ae2a5a6d37ee7404b066db/pulsar-common/src/main/java/org/apache/pulsar/common/policies/data/TenantOperation.java#L26
[5] https://github.com/apache/pulsar/blob/4c6026213b743a7f23ae2a5a6d37ee7404b066db/pulsar-broker-common/src/main/java/org/apache/pulsar/broker/authorization/AuthorizationService.java#L50
https://github.com/apache/pulsar/blob/4c6026213b743a7f23ae2a5a6d37ee7404b066db/pulsar-broker-common/src/main/java/org/apache/pulsar/broker/authorization/AuthorizationProvider.java#L46
[6] https://github.com/apache/pulsar/blob/7b65fab167ab776d78fbfd7482ca59fa2276f495/pulsar-broker/src/main/java/org/apache/pulsar/broker/admin/impl/PersistentTopicsBase.java#L320
https://github.com/apache/pulsar/blob/7b65fab167ab776d78fbfd7482ca59fa2276f495/pulsar-broker/src/main/java/org/apache/pulsar/broker/admin/impl/PersistentTopicsBase.java#L387







