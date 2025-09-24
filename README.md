# Sistema-Eventos-Java
// Projeto: Sistema de Cadastro e Notificação de Eventos
// Linguagem: Java (Console)
// Estrutura: arquivos separados listados abaixo. Compile com `javac` e execute `java Main`.

// -----------------------------
// File: Category.java
// -----------------------------
public enum Category {
    FESTA,
    ESPORTE,
    SHOW,
    CONFERENCIA,
    OUTRO;
}

// -----------------------------
// File: User.java
// -----------------------------
import java.io.Serializable;

public class User implements Serializable {
    private String username;
    private String fullName;
    private String email;
    private String city;

    public User(String username, String fullName, String email, String city) {
        this.username = username;
        this.fullName = fullName;
        this.email = email;
        this.city = city;
    }

    public String getUsername() { return username; }
    public String getFullName() { return fullName; }
    public String getEmail() { return email; }
    public String getCity() { return city; }

    @Override
    public String toString() {
        return fullName + " (" + username + ") - " + email + " - " + city;
    }
}

// -----------------------------
// File: Event.java
// -----------------------------
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.HashSet;
import java.util.Set;

public class Event {
    private static final DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");

    private int id;
    private String name;
    private String address;
    private Category category;
    private LocalDateTime dateTime; // horário de início
    private String description;
    private int durationHours; // opcional: duração em horas
    private Set<String> participants = new HashSet<>(); // usernames

    public Event(int id, String name, String address, Category category, LocalDateTime dateTime, String description, int durationHours) {
        this.id = id;
        this.name = name;
        this.address = address;
        this.category = category;
        this.dateTime = dateTime;
        this.description = description;
        this.durationHours = durationHours;
    }

    public int getId() { return id; }
    public String getName() { return name; }
    public String getAddress() { return address; }
    public Category getCategory() { return category; }
    public LocalDateTime getDateTime() { return dateTime; }
    public String getDescription() { return description; }
    public int getDurationHours() { return durationHours; }

    public void addParticipant(String username) { participants.add(username); }
    public void removeParticipant(String username) { participants.remove(username); }
    public boolean isParticipating(String username) { return participants.contains(username); }

    public boolean isPast() {
        return dateTime.plusHours(durationHours).isBefore(LocalDateTime.now());
    }

    public boolean isHappeningNow() {
        LocalDateTime now = LocalDateTime.now();
        return !now.isBefore(dateTime) && now.isBefore(dateTime.plusHours(durationHours));
    }

    public String participantsToString() {
        if (participants.isEmpty()) return "";
        return String.join(",", participants);
    }

    public String toDataLine() {
        // id|name|address|category|datetime|description|duration|p1,p2
        return id + "|" + escape(name) + "|" + escape(address) + "|" + category.name() + "|" + dateTime.format(fmt) + "|" + escape(description) + "|" + durationHours + "|" + participantsToString();
    }

    public static Event fromDataLine(String line) {
        try {
            String[] parts = line.split("\\|", -1);
            int id = Integer.parseInt(parts[0]);
            String name = unescape(parts[1]);
            String address = unescape(parts[2]);
            Category cat = Category.valueOf(parts[3]);
            LocalDateTime dt = LocalDateTime.parse(parts[4], fmt);
            String desc = unescape(parts[5]);
            int duration = Integer.parseInt(parts[6]);
            Event e = new Event(id, name, address, cat, dt, desc, duration);
            if (parts.length > 7 && !parts[7].isEmpty()) {
                String[] ps = parts[7].split(",");
                for (String p: ps) if (!p.isBlank()) e.participants.add(p.trim());
            }
            return e;
        } catch (Exception ex) {
            System.out.println("Erro ao ler linha de evento: " + ex.getMessage());
            return null;
        }
    }

    private static String escape(String s) {
        return s.replace("\\", "\\\\").replace("|", "\\|");
    }
    private static String unescape(String s) {
        return s.replace("\\|", "|").replace("\\\\", "\\");
    }

    @Override
    public String toString() {
        return "["+id+"] " + name + " (" + category + ") - " + dateTime.format(fmt) + " - " + address;
    }
}

// -----------------------------
// File: EventManager.java
// -----------------------------
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.File;
import java.io.FileReader;
import java.io.FileWriter;
import java.nio.file.Files;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

public class EventManager {
    private List<Event> events = new ArrayList<>();
    private int nextId = 1;
    private final String dataFile = "events.data";
    private static final DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");

    public EventManager() {
        load();
    }

    public List<Event> getAllEventsSorted() {
        return events.stream().sorted(Comparator.comparing(Event::getDateTime)).collect(Collectors.toList());
    }

    public List<Event> getUpcomingEvents() {
        LocalDateTime now = LocalDateTime.now();
        return events.stream().filter(e -> !e.isPast()).sorted(Comparator.comparing(Event::getDateTime)).collect(Collectors.toList());
    }

    public List<Event> getPastEvents() {
        return events.stream().filter(Event::isPast).sorted(Comparator.comparing(Event::getDateTime)).collect(Collectors.toList());
    }

    public Optional<Event> getById(int id) {
        return events.stream().filter(e -> e.getId() == id).findFirst();
    }

    public Event createEvent(String name, String address, Category cat, LocalDateTime dt, String desc, int duration) {
        Event e = new Event(nextId++, name, address, cat, dt, desc, duration);
        events.add(e);
        return e;
    }

    public void save() {
        try (BufferedWriter w = new BufferedWriter(new FileWriter(dataFile))) {
            for (Event e: events) {
                w.write(e.toDataLine());
                w.newLine();
            }
        } catch (Exception ex) {
            System.out.println("Erro ao salvar eventos: " + ex.getMessage());
        }
    }

    public void load() {
        events.clear();
        File f = new File(dataFile);
        if (!f.exists()) return;
        try (BufferedReader r = new BufferedReader(new FileReader(f))) {
            String line;
            int maxId = 0;
            while ((line = r.readLine()) != null) {
                if (line.isBlank()) continue;
                Event e = Event.fromDataLine(line);
                if (e != null) {
                    events.add(e);
                    if (e.getId() > maxId) maxId = e.getId();
                }
            }
            nextId = maxId + 1;
        } catch (Exception ex) {
            System.out.println("Erro ao carregar eventos: " + ex.getMessage());
        }
    }

}

// -----------------------------
// File: Main.java
// -----------------------------
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.List;
import java.util.Optional;
import java.util.Scanner;

public class Main {
    private static final DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        EventManager manager = new EventManager();
        User currentUser = null;

        System.out.println("Bem-vindo ao Sistema de Eventos (Console)");

        while (true) {
            System.out.println();
            System.out.println("Usuário atual: " + (currentUser == null ? "(nenhum)" : currentUser));
            System.out.println("1. Registrar usuário");
            System.out.println("2. Entrar com usuário (login por username)");
            System.out.println("3. Listar eventos (ordenados por horário)");
            System.out.println("4. Cadastrar evento");
            System.out.println("5. Ver evento (detalhes / participar / cancelar)");
            System.out.println("6. Meus eventos confirmados");
            System.out.println("7. Eventos passados");
            System.out.println("8. Salvar e sair");
            System.out.print("Escolha: ");
            String opt = sc.nextLine().trim();

            switch (opt) {
                case "1":
                    currentUser = registerUser(sc);
                    break;
                case "2":
                    currentUser = loginUser(sc);
                    break;
                case "3":
                    listEvents(manager);
                    break;
                case "4":
                    if (currentUser == null) {
                        System.out.println("Faça login ou registre-se para criar eventos.");
                        break;
                    }
                    createEventInteractive(sc, manager);
                    break;
                case "5":
                    if (currentUser == null) {
                        System.out.println("Faça login ou registre-se para participar de eventos.");
                        break;
                    }
                    viewEventInteractive(sc, manager, currentUser);
                    break;
                case "6":
                    if (currentUser == null) { System.out.println("Faça login primeiro."); break; }
                    myConfirmedEvents(manager, currentUser);
                    break;
                case "7":
                    pastEvents(manager);
                    break;
                case "8":
                    manager.save();
                    System.out.println("Salvo em events.data. Saindo...");
                    sc.close();
                    return;
                default:
                    System.out.println("Opção inválida.");
            }
        }
    }

    private static User registerUser(Scanner sc) {
        System.out.print("Username: "); String u = sc.nextLine().trim();
        System.out.print("Nome completo: "); String n = sc.nextLine().trim();
        System.out.print("Email: "); String e = sc.nextLine().trim();
        System.out.print("Cidade: "); String c = sc.nextLine().trim();
        User user = new User(u, n, e, c);
        System.out.println("Usuário registrado: " + user);
        return user;
    }

    private static User loginUser(Scanner sc) {
        System.out.print("Digite o username: "); String u = sc.nextLine().trim();
        // Not persisting users: login simply creates a placeholder user with that username
        System.out.print("Nome completo (apenas para exibição): "); String n = sc.nextLine().trim();
        System.out.print("Email: "); String e = sc.nextLine().trim();
        System.out.print("Cidade: "); String c = sc.nextLine().trim();
        User user = new User(u, n, e, c);
        System.out.println("Logado como: " + user);
        return user;
    }

    private static void listEvents(EventManager manager) {
        List<Event> all = manager.getAllEventsSorted();
        if (all.isEmpty()) { System.out.println("Nenhum evento cadastrado."); return; }
        System.out.println("Eventos (ordenados por horário):");
        for (Event e : all) {
            String status = e.isHappeningNow() ? "[OCORRENDO]" : (e.isPast() ? "[PASSOU]" : "[AGENDADO]");
            System.out.println(e + " " + status);
        }
    }

    private static void createEventInteractive(Scanner sc, EventManager manager) {
        try {
            System.out.print("Nome do evento: "); String name = sc.nextLine().trim();
            System.out.print("Endereço: "); String addr = sc.nextLine().trim();
            System.out.println("Categorias: ");
            for (Category c : Category.values()) System.out.println(" - " + c.name());
            System.out.print("Escolha categoria (ex: FESTA): "); String catS = sc.nextLine().trim().toUpperCase();
            Category cat = Category.valueOf(catS);
            System.out.print("Horário (yyyy-MM-dd HH:mm): "); String dtS = sc.nextLine().trim();
            LocalDateTime dt = LocalDateTime.parse(dtS, fmt);
            System.out.print("Duração em horas (inteiro, ex: 3): "); int dur = Integer.parseInt(sc.nextLine().trim());
            System.out.print("Descrição: "); String desc = sc.nextLine().trim();
            Event e = manager.createEvent(name, addr, cat, dt, desc, dur);
            System.out.println("Evento criado: " + e);
        } catch (Exception ex) {
            System.out.println("Erro ao criar evento: " + ex.getMessage());
        }
    }

    private static void viewEventInteractive(Scanner sc, EventManager manager, User user) {
        System.out.print("Digite o id do evento: ");
        try {
            int id = Integer.parseInt(sc.nextLine().trim());
            Optional<Event> oe = manager.getById(id);
            if (oe.isEmpty()) { System.out.println("Evento não encontrado."); return; }
            Event e = oe.get();
            System.out.println("Detalhes:\n" + e);
            System.out.println("Descrição: " + e.getDescription());
            System.out.println("Duração (h): " + e.getDurationHours());
            System.out.println("Participantes: " + (e.participantsToString().isEmpty() ? "Nenhum" : e.participantsToString()));
            System.out.println("Status: " + (e.isHappeningNow() ? "OCORRENDO" : (e.isPast() ? "PASSOU" : "AGENDADO")));
            System.out.println("1. Confirmar presença");
            System.out.println("2. Cancelar presença");
            System.out.println("3. Voltar");
            System.out.print("Escolha: ");
            String op = sc.nextLine().trim();
            switch (op) {
                case "1":
                    if (e.isPast()) { System.out.println("Não é possível confirmar presença em evento já passado."); }
                    else { e.addParticipant(user.getUsername()); System.out.println("Presença confirmada."); }
                    break;
                case "2":
                    e.removeParticipant(user.getUsername()); System.out.println("Presença cancelada (se havia).");
                    break;
                default:
                    break;
            }
        } catch (Exception ex) {
            System.out.println("Entrada inválida: " + ex.getMessage());
        }
    }

    private static void myConfirmedEvents(EventManager manager, User user) {
        List<Event> my = manager.getAllEventsSorted().stream().filter(e -> e.isParticipating(user.getUsername())).collect(java.util.stream.Collectors.toList());
        if (my.isEmpty()) { System.out.println("Você não confirmou presença em nenhum evento."); return; }
        System.out.println("Eventos com sua presença confirmada:");
        for (Event e: my) {
            System.out.println(e + " - Status: " + (e.isHappeningNow() ? "OCORRENDO" : (e.isPast() ? "PASSOU" : "AGENDADO")));
        }
    }

    private static void pastEvents(EventManager manager) {
        List<Event> past = manager.getPastEvents();
        if (past.isEmpty()) { System.out.println("Nenhum evento passado."); return; }
        System.out.println("Eventos que já ocorreram:");
        for (Event e: past) System.out.println(e + " - ocorrido em: " + e.getDateTime().format(fmt));
    }
}

// -----------------------------
// README (instruções rápidas)
// -----------------------------
/*
Como compilar e executar (linha de comando):
1. Salve cada bloco em um arquivo com o nome indicado (Category.java, User.java, Event.java, EventManager.java, Main.java).
2. Compile: javac *.java
3. Execute: java Main

O programa carrega automaticamente events.data (se existir) e salva neste mesmo arquivo ao escolher "Salvar e sair".
Formato do arquivo events.data: cada linha corresponde a um evento e segue o formato:
id|name|address|category|yyyy-MM-dd HH:mm|description|durationHours|participant1,participant2

*/
