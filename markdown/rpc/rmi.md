# rmi
---
## 一 RMI
### 1.1 接口
RMI接口必须直接或间接扩展java.rmi.Remote接口<br>
RMI接口中方法必须在throws语句中声明抛出java.rmi.RemoteException

    @Data
    class Mumu implements Serializable {
        private static final Long serialVersionUID = 1l;
        private String name;
        private Integer age;
    }
    
    public interface MumuService extends Remote {
        Mumu getMumuByName(String name) throws RemoteException;
    }
### 1.2 实现
可以使用java.rmi.server.RemoteServer和它的子类来实现远程调用，下例中使用UnicastRemoteObject来实现

    @Slf4j
    public class MumuServiceImpl extends UnicastRemoteObject implements MumuService {
    
        private static final Long serialVersionUID = 1l;
    
        public Mumu getMumuByName(String name) throws RemoteException {
            return Mumu.builder().name(name).age(18).build();
        }
    }