Huawei RFC9234
==============

За пример возьмем доку - https://support.huawei.com/enterprise/en/doc/EDOC1100334538/a0c4ad3c/understanding-bgp

Поддержка данного RFC судя по всему начинается с R023, а значит маршрутизаторы старики например NE20E остаются в пролете.
Опишем роли:

- Provider: The local AS is the transit provider of the remote AS and can advertise any available routes to a customer.
- Route server (RS): The local AS is a route server and can advertise any available routes to an RS-client (remote AS).
- RS-client: The local AS is an RS-client and can advertise any routes learned from a customer or locally originated routes to an RS.
- Customer: The local AS is a transit customer of the remote AS and can advertise any routes learned from a customer or locally originated routes to a provider.
- Lateral-peer: If the local AS and the remote AS have a lateral peer relationship, any routes learned from a customer or locally originated routes can be advertised to the peer end.
- Sibling: If the local and remote ASs have the sibling peer relationship, they can advertise all routes (both customer and non-customer routes) to each other.

Дополнения от себя: Sibling - чем похож на тип узла локального или не локального, но типо роут-рефлектора. Lateral-peer - честно говоря не особо нашёл ему применение особо на узлах в качестве бордера.

Рассмотрим пример конфигурации на железе:
::

    bgp XXXXX
     ipv4-family unicast
      peer X.X.X.X role 
