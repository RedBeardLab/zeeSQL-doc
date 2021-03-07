# zeeSQL commits to backward compatibility

zeeSQL commits to backward compatibility.

Software written to run on zeeSQL version `n` should be able to run, unchanged on any version greater than `n`.

Unfortunately, committing so strictly to backward compatibilities, also means that the APIs that `zeeSQL` exposes are almost immutable, this is not sustainable.

Hence we offer a very reasonable middle ground.

## Namespacing the command

Each command of `zeeSQL` is namespaced also with its major version.

Now, `zeeSQL` is at version `1`, hence all the commands available in `ZEESQL`, like `ZEESQL.CREATE_DB` or `ZEESQL.EXEC`, are also available in `ZEESQL.V1`.
In the specific case, `ZEESQL.CRATE_DB` is available also as `ZEESQL.V1.CREATE_DB` and `ZESQL.EXEC` is also available as `ZEESQL.V1.EXEC`.
This, of course, holds for every single command exposed by `zeeSQL`.

When `zeeSQL` version `2` will be released, all the commands in `ZEESQL.V1` will stay exactly the same.
The code won't change, and they will map one to one to the commands in version `1`.

However, when version `2` of `zeeSQL` will be available, all the commands without the version namespace, `ZEESQL.CREATE_DB` (for instance) will be the same as `ZEESQL.V2.CREATE_DB` not of `ZEESQL.V1.CREATE_DB`.

This schema allows us to move forward with the APIs, fixing mistakes, but will also provide stable backward compatibility guarantees for our users.

## Which version of the command to use, with or without version namespace?

What specific command version you should use in your project depends.

If you are not so interested in updates and new features, and the project might become legacy in your organization, in this case, it would be better to use the commands with the namespace.
In this way, you can be sure that your code will always work and that you can update `zeeSQL` for performance or security reasons.

If you are experimenting, are interested in more features, or are betting a lot on `zeeSQL` tracking the version without the namespace is a good idea as well.
This might involve more work in the future, but it will also give access to more features.

## Conclusions

Overall, we hope that this commitment to backward compatibility, show how much we care about our users and that our priorities are about selling good software that simplifies our users' life.

Not committing to backward compatibility would have been a short-term choice to quickly get more revenues at the cost of losing the trust of the community.


