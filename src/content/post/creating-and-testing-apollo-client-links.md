---
title: "Creating and Testing Apollo Client Links"
description: "This post is for testing the draft post functionality"
publishDate: "24 Oct 2023"
tags: ["apollo client", "graphql", "jest", "testing"]
---

When using Apollo Client, there might come a time when you want to inspect or make some changes to your GraphQL requests. This is where Apollo Link comes into play. The library sets up a chain of actions for each GraphQL operation the client executes.

What exactly is a link, and how would we create and test one? Let's get into it and see what's going on.

## Understanding the Concept of a Link

In order to grasp the concept of a link in the context of Apollo Client, let's start with what the docs tell us:

> The **Apollo Link** library helps you customize the flow of data between Apollo Client and your GraphQL server. You can define your client's network behavior as a chain of **link** objects that execute in a sequence. Each link should represent either a self-contained modification to a GraphQL operation or a side effect (such as logging). [Source](https://www.apollographql.com/docs/react/api/link/introduction/)

When Apollo Client initiates a request, the request traverses through the chain of links until it reaches a _terminating link_. Typically, the terminating link is responsible for executing the actual fetch to the GraphQL server and managing the subsequent response. The response then travels back through the chain in reverse order.

```jsx
// The `from` function combines an array of individual links into a link chain
const client = new ApolloClient({
  link: ApolloLink.from([errorLink, myCustomLink, httpLink]),
  ...
});

// alternatively, you can also use concat like so
const client = new ApolloClient({
  link: errorLink.concat(httpLink)
  ...
});
```

## Creating a Link

There are two types of links: [stateless and stateful](https://www.apollographql.com/docs/react/api/link/introduction/#link-types). A link is an instance of the ApolloLink class (or a subclass of it) and each link requires the definition of a request handler: `request(operation, forward)`.

Let's take a quick look at those two request handler params:

- **operation** - this is all the juicy info about the current, well, _operation_ - the entirety of the query and what is attached to it. Here we can easily access `operation.query` and `operation.variables` to understand information about the request, as well as the rest of the `context` for the request via `operation.getContext()`

- **forward** - this is a function that takes the operation and _forwards_ it to the next link, like so: `forward(operation)`

### Handling Responses in the Link Chain

Now that we've successfully forwarded the operation along the link chain, it's time to handle the response as it journeys back through the chain. To accomplish this, we need to establish an observer and subscribe to the response.

Fortunately, this process is quite straightforward. Apollo Client provides the Observable class for precisely this purpose. You can employ this class to set up an observer subscription. Here's how it's done:

```jsx
import { Observable } from '@apollo/client';

// the required request handler
request(operation, forward) {
  return new Observable((observer) => {
 const subscription = forward(operation).subscribe({
   next: (result) => {
     observer.next(result);
     observer.complete();
   },
   error: (error) => observer.error(error),
   complete: () => observer.complete(),
 });

 return () => {
      subscription.unsubscribe();
    }
  });
}
```

In the code snippet above, we create a new Observable instance within the request function. This Observable instance facilitates the subscription to the response. We subscribe to the response by forwarding the operation and defining handlers: `next`, which receives the data from a successful operation attempt, `error`, and `complete`, which signals we are done with handling the response.

While the code above represents a straightforward way to set up a subscription, it really doesn't do anything yet. The real magic happens when we add custom logic into this process. For instance, if we set a `startTime` in our request handler before forwarding the operation, we can then utilize our observer logic to calculate total request time:

```jsx
request(operation, forward) {
  // set the startTime before we forward the operation
  operation.setContext({
    startTime: Date.now(),
  });

  return new Observable((observer) => {
    const subscription = forward(operation).subscribe({
      next: (result) => {
        const { operationName } = operation;
        const startTime = operation.getContext().startTime;

        if (operationName && startTime) {
          // calculate total request time in our observer
          logRequestTime(
            operationName,
            Date.now() - startTime
          );
        }

        observer.next(result);
        observer.complete();
      },
      error: (error) => observer.error(error),
      complete: () => observer.complete(),
    });

    return () => {
      subscription.unsubscribe();
    }
  });
}
```

Here, we perform additional logic by recording the request time before forwarding the result to the observer. This flexibility enables you to intercept and manipulate data seamlessly within Apollo, adding a powerful tool to your toolkit.

## Testing Apollo Links

When testing an Apollo Link, it's useful to write a separate simple link which acts as a terminating link. Then, create a composed link with the link to test, with the terminating link being our mock link. This test link should be simple and respond in a predetermined the way in the scope of our testing.

Here's an example mockLink:

```jsx
const mockLink = new ApolloLink((operation) => {
	const context = operation.getContext();
	const { mockError, mockResponse } = context || {};

	return new Observable((observer) => {
		if (mockError) {
			observer.error(mockError);
			return;
		}

		if (mockResponse) {
			observer.next({ data: mockResponse });
			observer.complete();
			return;
		}

		observer.error(new Error("mockLink: You must provide mockError or mockResponse"));
	});
});
```

Within the mockLink function, we inspect the operation's context to check for the presence of `mockError` or `mockResponse`. We construct an Observable which, depending on the context either emits a mock error or a mock response wrapped in the expected data structure. In case neither is provided, the mockLink generates an error to ensure that test cases explicitly define the expected behavior.

This link then sends data or an error back down the chain (in our case, back to the link we are testing).

## A Simple Link to Test

Let's see this in action to make more sense of it. Imagine our link `OperationHistoryLink` does exactly that: keeps an array of our operations in order as we execute them. We can create a stateful link to handle this for us:

```jsx
import { ApolloLink } from "@apollo/client";

// stateful link
class OperationHistoryLink extends ApolloLink {
	operationHistory = [];

	request(operation, forward) {
		this.operationHistory.push(operation);
		return forward(operation);
	}
}
```

To test this, we must execute a query, then assert the operation is added to the array when it succeeds.

```jsx
import { execute } from "@apollo/client";

describe("OperationHistoryLink", () => {
	it("adds operation to the operationHistory array on success", () => {
		const operationHistoryLink = new OperationHistoryLink();
		const composedLink = operationHistoryLink.concat(mockLink);

		return new Promise((resolve) => {
			const operation = {
				query: "some query",
				context: {
					mockResponse: { foo: "bar" },
				},
			};

			execute(composedLink, operation).subscribe({
				next: ({ data }) => {
					expect(data.foo).toBe("bar"); // mockResponse is handled by mockLink
					expect(operationHistoryLink.operationHistory).toHaveLength(1);
					expect(operationHistoryLink.operationHistory[0].query).toEqual(operation.query);
					resolve();
				},
				error: () => {
					// optionally reject or throw here, we shouldn't get an error
				},
				complete: () => {},
			});
		});
	});
});
```

A few things to note about our test:

- To trigger our GraphQL operation, we use the `execute` function provided by Apollo Client. This function begins the process of sending the operation down the link chain, creating an observer subscription with the same signature as we used in our link itself (`next`, `error`, `complete`)
- To ensure that our test has time to execute, we use a `Promise`. This allows us to await the resolution of the Promise, ensuring that our test assertions are carried out only after the GraphQL operation has completed its execution
- Our assertions live inside our `next` subscription callback. Of course, when our data comes back from our mockLink, we can expect it to match what we sent in the context. However, the main thing we are asserting here is the `operationHistory` in our `OperationHistoryLink` has a length of 1. If that is true, we know our link is working as intended!

```sh
OperationHistoryLink
    ✓ adds operation to the operationHistory array on success (3 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        2.141 s
```

Success! We have now successfully created a custom link, and written a test to assert its functionality.

Want to try it out yourself? [You can view all the code here](https://gist.github.com/jasongaare/5891bff560ceee5b0b8b5cb70f629a1d)

---

[Originally posted on DEV Community](https://dev.to/companycam/creating-and-testing-apollo-client-links-cca)
