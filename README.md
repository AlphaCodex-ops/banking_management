# banking_management_code

import java.io.*;
import java.util.*;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class BankDatabaseManager {

    static class Account implements Serializable {
        String id, n, ph, em, addr, pw;
        double bal;
        public Account(String id, String n, double bal, String ph, String em, String addr) {
            this.id=id; this.n=n; this.bal=bal; this.ph=ph; this.em=em; this.addr=addr; this.pw=id;
        }
        public String getId() {return id;}
        public String getN() {return n;}
        public String getPh() {return ph;}
        public String getPw() {return pw;}
        public double getBal() {return bal;}
        public void setPw(String pw) {this.pw=pw;}
        public void dep(double amt) {bal+=amt;}
        public void wd(double amt) {bal-=amt;}
        public String getInfo() {
            return String.format("ID: %s\nName: %s\nBal: %.2f\nPh: %s\nEm: %s\nAddr: %s", id,n,bal,ph,em,addr);
        }
    }

    static class Txn implements Serializable {
        String tid, fId, tId, tp, ts; double amt;
        public Txn(String tid, String fId, String tId, double amt, String tp, String ts) {
            this.tid=tid; this.fId=fId; this.tId=tId; this.amt=amt; this.tp=tp; this.ts=ts;
        }
    }

    static class DataMngr {
        List<Account> accs=new ArrayList<>();
        List<Txn> txns=new ArrayList<>();
        String acFile="accounts.dat", txnFile="transactions.dat";
        int lastN=1000;
        public void loadAll() {
            try(ObjectInputStream ois=new ObjectInputStream(new FileInputStream(acFile))) {
                accs=(List<Account>)ois.readObject(); updId();
            } catch(Exception e) {System.out.println("No accounts found.");}
            try(ObjectInputStream ois=new ObjectInputStream(new FileInputStream(txnFile))) {
                txns=(List<Txn>)ois.readObject();
            } catch(Exception e) {System.out.println("No txns found.");}
        }
        public void saveAll() {
            try(ObjectOutputStream oos=new ObjectOutputStream(new FileOutputStream(acFile))) {
                oos.writeObject(accs);
            } catch(Exception e) {System.out.println("Save accounts fail.");}
            try(ObjectOutputStream oos=new ObjectOutputStream(new FileOutputStream(txnFile))) {
                oos.writeObject(txns);
            } catch(Exception e) {System.out.println("Save txns fail.");}
        }
        private void updId() {
            for(Account a: accs) {
                if(a.id.startsWith("C")) {
                    try {
                        int num=Integer.parseInt(a.id.substring(1));
                        if(num>lastN) lastN=num;
                    } catch(Exception ignored){}
                }
            }
        }
        public String genId() {return "C"+(++lastN);}
        public void addAcc(Account a) {accs.add(a);}
        public Account findAcc(String id) {
            for(Account a:accs) if(a.getId().equalsIgnoreCase(id)) return a;
            return null;
        }
        public void addTxn(Txn t) {txns.add(t);}
        public List<Txn> getTxnsById(String id) {
            List<Txn> ans=new ArrayList<>();
            for(Txn t:txns) if(t.fId.equalsIgnoreCase(id)||t.tId.equalsIgnoreCase(id)) ans.add(t);
            return ans;
        }
        public List<Account> getAllAccs() {return accs;}
    }

    static class TxnMngr {
        DataMngr dm; static int tCount=1000;
        public TxnMngr(DataMngr dm) {this.dm=dm;}
        String genTid() {return "T"+(++tCount);}
        String now() {
            LocalDateTime n=LocalDateTime.now();
            return n.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        }
        public boolean dep(Account a, double amt) {
            if(amt<=0) return false;
            a.dep(amt);
            dm.addTxn(new Txn(genTid(),"BANK",a.id,amt,"DEP",now()));
            return true;
        }
        public boolean wd(Account a, double amt) {
            if(amt<=0||amt>a.bal) return false;
            a.wd(amt);
            dm.addTxn(new Txn(genTid(),a.id,"BANK",amt,"WD",now()));
            return true;
        }
        public boolean tf(Account f, Account t, double amt) {
            if(amt<=0||amt>f.bal) return false;
            f.wd(amt); t.dep(amt);
            dm.addTxn(new Txn(genTid(),f.id,t.id,amt,"TF",now()));
            return true;
        }
    }

    static class AdminInt {
        DataMngr dm; Scanner sc=new Scanner(System.in);
        public AdminInt(DataMngr dm) {this.dm=dm;}
        public void menu() {
            while(true) {
                System.out.println("\nAdmin:1.Crt 2.VAll 3.Srch 4.Exit");
                int ch=Utils.gInt();
                switch(ch) {
                    case 1:crtAcc();break;
                    case 2:vAll();break;
                    case 3:srch();break;
                    case 4:return;
                    default:System.out.println("Inv!");break;
                }
            }
        }
        void crtAcc() {
            System.out.print("Name: "); String n=sc.nextLine();
            System.out.print("Dep: "); double d=Utils.gD();
            System.out.print("Ph: "); String ph=sc.nextLine();
            System.out.print("Em: "); String em=sc.nextLine();
            System.out.print("Addr: "); String addr=sc.nextLine();
            String id=dm.genId();
            Account a=new Account(id,n,d,ph,em,addr);
            dm.addAcc(a); dm.saveAll();
            System.out.println("Created ID:"+id);
        }
        void vAll() {
            System.out.println("\nAccs:");
            for(Account a:dm.getAllAccs())
                System.out.printf("%s - %s - %.2f - %s\n",a.id,a.n,a.bal,a.ph);
        }
        void srch() {
            System.out.print("CID: ");
            String id=sc.nextLine();
            Account a=dm.findAcc(id);
            if(a==null) System.out.println("No");
            else System.out.println(a.getInfo());
        }
    }

    static class CustInt {
        DataMngr dm; Account a; Scanner sc=new Scanner(System.in); TxnMngr tm;
        public CustInt(DataMngr dm, Account a) {this.dm=dm;this.a=a; tm=new TxnMngr(dm);}
        public void menu() {
            System.out.println("Welcome "+a.n);
            while(true) {
                System.out.println("\nCust:1.VAcc 2.Dep 3.Wd 4.Tf 5.VTx 6.Pw 7.Logout");
                int c=Utils.gInt();
                switch(c) {
                    case 1:System.out.println(a.getInfo());break;
                    case 2:dep();break;
                    case 3:wd();break;
                    case 4:tf();break;
                    case 5:vTx();break;
                    case 6:pwChg();break;
                    case 7:dm.saveAll();return;
                    default:System.out.println("Inv!");break;
                }
            }
        }
        void dep() {
            System.out.print("Amt: "); double d=Utils.gD();
            if(tm.dep(a,d)) {System.out.println("Dep success "+a.bal); dm.saveAll();}
            else System.out.println("Fail");
        }
        void wd() {
            System.out.print("Amt: "); double d=Utils.gD();
            if(tm.wd(a,d)) {System.out.println("Wd success "+a.bal); dm.saveAll();}
            else System.out.println("Fail");
        }
        void tf() {
            System.out.print("To CID: "); String id=sc.nextLine();
            Account t=dm.findAcc(id);
            if(t==null) {System.out.println("No"); return;}
            System.out.print("Amt: "); double d=Utils.gD();
            if(tm.tf(a,t,d)) {System.out.println("Tf success "+a.bal); dm.saveAll();}
            else System.out.println("Fail");
        }
        void vTx() {
            List<Txn> l=dm.getTxnsById(a.id);
            if(l.isEmpty()) System.out.println("No txns");
            else for(Txn t:l) System.out.printf("%s %s->%s %.2f %s\n",t.tid,t.fId,t.tId,t.amt,t.ts);
        }
        void pwChg() {
            System.out.print("Cur PW: "); String c=sc.nextLine();
            if(!a.getPw().equals(c)) {System.out.println("Wrong"); return;}
            System.out.print("New PW: "); String np=sc.nextLine();
            System.out.print("Conf NP: "); String cn=sc.nextLine();
            if(!np.equals(cn)) {System.out.println("No match"); return;}
            a.setPw(np); dm.saveAll(); System.out.println("PW changed");
        }
    }

    static class Utils {
        static Scanner sc=new Scanner(System.in);
        public static int gInt() {
            try {return Integer.parseInt(sc.nextLine());}
            catch(Exception e) {return -1;}
        }
        public static double gD() {
            try {return Double.parseDouble(sc.nextLine());}
            catch(Exception e) {return -1;}
        }
    }

    public static void main(String[] args) {
        DataMngr dm=new DataMngr();
        dm.loadAll();
        Scanner sc=new Scanner(System.in);
        while(true) {
            System.out.println("1.Admin 2.Cust 3.Exit");
            int c=Utils.gInt();
            switch(c) {
                case 1:
                    System.out.print("User: ");
                    String u=sc.nextLine();
                    System.out.print("PW: ");
                    String p=sc.nextLine();
                    if("admin".equals(u) && "password123".equals(p)) new AdminInt(dm).menu();
                    else System.out.println("Invalid");
                    break;
                case 2:
                    System.out.print("ID: ");
                    String id=sc.nextLine();
                    System.out.print("PW: ");
                    String pw=sc.nextLine();
                    Account a=dm.findAcc(id);
                    if(a!=null && pw.equals(a.getPw())) new CustInt(dm,a).menu();
                    else System.out.println("Invalid");
                    break;
                case 3:
                    System.out.println("Bye!");
                    dm.saveAll();
                    System.exit(0);
                default:
                    System.out.println("No choice");
            }
        }
    }
}
